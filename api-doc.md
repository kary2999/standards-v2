---
title: "接口文档规范"
version: "1.3.0"
last_modified: "2026-05-20"
source: "技术规范.2026.05.20 / 接口文档示例.md"
---

# HTTP 接口规范速查（v1.3.0）

> 适用范围：内部所有后端服务对外暴露的 HTTP/HTTPS 接口
> 风格：RESTful，**仅使用 GET 与 POST**，**所有参数统一通过 JSON Body 传输（POST）**，URL 中禁用 query string

---

## 1. 核心原则（必读）

1. **GET 极简**：仅用于按路径参数取单资源，无 query string，无 Body
2. **POST 承载一切**：列表、分页、搜索、统计、创建、更新、删除、状态流转、批量、导出
3. **禁用 query string**：URL 中不得出现 `?key=value`，可变参数全进 Body
4. **前端 URL 不含版本号**：`/api/v{n}` 由网关在转发时拼接，前端零感知

---

## 2. URL 规范

### 前端可见 URL（对外契约）

```
https://{host}/{service}/{resource}[/{id}][/{sub-resource}][/{action}]
```

| 部分 | 规则 | 示例 |
|---|---|---|
| `service` | 全小写、短横线分隔 | `user-center` |
| `resource` | 复数、小写、短横线分隔 | `orders`、`pay-channels` |
| `id` | 路径参数，URL 安全字符 | `1001` |
| `action` | 动词短语 | `list`、`search`、`cancel`、`batch-export` |

### 禁止项

- URL 中不得有 `?`、`&`、`=`
- 前端 URL 中不得有 `api`、`v1`、`v2` 等版本段
- 不得用 `/getUserList`、`/createOrder` 等动词接口名
- 不得使用大写、下划线、空格
- 路径参数不得含敏感信息（手机号、证件号、Token）
- **不得在路径里体现业务主域名**（`earn` / `spot` / `perp` / `c2c` 等）。主域由 `service` 段或网关路由承载，`resource` 只写资源本身：
  - ❌ `/earn/orders`、`/api/v1/earn/products`、`/spot/trades`
  - ✅ `/api/v1/orders`、`/api/v1/products`（属于哪个主域由网关按 `service` 路由决定）

### 网关→后端（内部路由）

```
http://{backend-pool}/api/v{major}/{resource}[/{id}][/{action}]
```

版本切换、灰度、回滚全部通过**网关路由配置**完成，前端不改代码。

---

## 3. GET / POST 使用约定

### GET：仅用于按路径取单资源

```
GET /order/orders/1001          → 订单详情
GET /order/orders/1001/summary  → 订单摘要（固定子资源）
GET /users/me                   → 当前登录用户
```

必须满足：幂等、无副作用、无 query string、无 Body、`Cache-Control: no-store`

### POST：其他一切交互

| 场景 | URL | 替代的传统方法 |
|---|---|---|
| 列表/分页查询 | `POST /orders/list` | GET 列表 |
| 复杂搜索 | `POST /orders/search` | GET 列表 |
| 统计/聚合 | `POST /orders/stat` | GET |
| 创建 | `POST /orders` | POST |
| 更新（全量/局部） | `POST /orders/1001` | PUT/PATCH |
| 删除（逻辑删除） | `POST /orders/1001/delete` | DELETE |
| 状态流转 | `POST /orders/1001/cancel` | PATCH |
| 批量操作 | `POST /orders/batch-update` | PATCH/DELETE |
| 导出触发 | `POST /orders/export` | GET |

> 即便只有一个参数，也用 POST + Body，不退化为 GET + query。

---

## 4. 请求规范

### 通用请求头

| Header | 是否必填 | 说明 |
|---|---|---|
| `Content-Type` | POST 必填 | `application/json; charset=utf-8` |
| `X-Trace-Id` | 建议 | 全链路追踪 ID，服务端必须在响应 `trace_id` 字段原样回传 |
| `Authorization` | 见鉴权章节 | 鉴权信息 |
| `Cache-Control` | 建议 | 列表/敏感接口固定 `no-store` |

### POST 请求体结构

- 根节点必须是对象 `{}`，不得是数组或基本类型
- 字段命名：**统一 snake_case（下划线），禁止驼峰**
- 可选字段缺失时省略，不传 `null` 占位
- 请求体 ≤ 2MB，超过走文件上传

### 列表/搜索类请求分块（强制）

```json
{
  "pagination": {
    "page": 1,
    "size": 20,
    "sort_by": "create_time",
    "order": "desc",
    "with_column": 1
  },
  "filters": {
    "status": ["PAID"],
    "keyword": "苹果"
  }
}
```

> `pagination` 与 `filters` **必须分块**，不得混在同一层级。
> `filters` 内不得出现保留字段：`page` / `size` / `sort_by` / `order` / `cursor` / `with_column`。

### `pagination` 对象字段

| 字段 | 类型 | 必填 | 默认 | 说明 |
|---|---|---|---|---|
| `page` | int | 否 | `1` | **1-indexed**；缺失或 ≤ 0 时服务端按 `1` 处理；与 `cursor` 互斥 |
| `size` | int | 否 | `20` | 每页条数，最大 `100`，超出由服务端裁剪 |
| `sort_by` | string | 否 | 业务定 | 排序字段（snake_case），服务端做白名单校验 |
| `order` | string | 否 | `desc` | `asc` / `desc` |
| `with_column` | int | 否 | `0` | `0`=不需要列定义，`1`=需要列定义（见 §6.3） |
| `cursor` | string | 否 | — | 游标分页：上次响应返回的 `next_cursor`，首次不传；与 `page` 互斥 |

### 服务端规范化（强制）

| 字段 | 缺失/null | 非法值 | 越界 |
|---|---|---|---|
| `page` | 按 `1` 处理 | ≤ 0 / 非整数 → 按 `1` | `page * size > 10000` → 返回 `COMMON_COMMON_002` 并提示用 cursor |
| `size` | 按 `20` 处理 | ≤ 0 / 非整数 → 按 `20` | > 100 → 裁剪为 `100`，`data.warnings` 中提示 |
| `sort_by` | 业务默认 | 不在白名单 → 业务默认 + `data.warnings` 提示 | — |
| `order` | `desc` | 非 asc/desc → `desc` | — |
| `with_column` | `0` | 非 0/1 → `0` | — |
| `cursor` | 视为首次请求 | 解析失败/已过期 → `COMMON_COMMON_002` | — |

> Spring 项目：设置 `spring.data.web.pageable.one-indexed-parameters=true` 或在入口做显式转换，确保 `page` 1-indexed 契约成立。

### 幂等性要求（写接口）

- 创建类：调用方传 `idempotency_key`（Body 根节点，字符串），服务端 24h 内对同一 key 返回首次结果
- **`idempotency_key` 与 `trace_id` 是两个不同概念，绝不复用**
- 状态流转类：目标状态相同视为幂等成功

---

## 5. 响应规范

### 统一外壳（强制）

```json
{
  "code": "0",
  "msg": "success",
  "data": {},
  "trace_id": "8c41e0a4-77f2-4d88-9a9d-1f1a9c7df222",
  "timestamp": "2026-04-28T10:15:30.123+08:00"
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `code` | string | 是 | `"0"` 表示成功；非 `"0"` 为业务失败 |
| `msg` | string | 是 | 成功为 `"success"`，失败为可直接展示的提示 |
| `data` | object/array | 是 | **永远是 `{}` 或 `[]`，禁止为 `null`** |
| `trace_id` | string | 是 | 与请求头 `X-Trace-Id` 一致，原样回传 |
| `timestamp` | string | 是 | ISO 8601 带时区，如 `"2026-04-28T10:15:30.123+08:00"` |

### `data` 的取值约定（强制）

| 接口类型 | `data` 内容 |
|---|---|
| **列表/分页/搜索** | `{ "list": [], "pagination": { ... } }`；空结果时 `list: []`，不是 `null` |
| 单资源详情 | 命中返回资源对象；未命中返回业务错误码，`data: {}` |
| 创建/更新/状态流转 | 至少返回受影响主键，如 `{ "order_id": "1003" }` |
| 删除（软删除） | `data: {}` |
| 批量操作 | `{ "success_ids": [], "failures": [] }`；空时也用 `[]` |
| 校验失败 | `{ "errors": [{ "field": "xxx", "reason": "..." }] }` |
| 其他业务失败 | `data: {}` |

> `data: null` 在任何场景下都不允许出现。

### HTTP 状态码

仅用以下少量状态码，**业务错误一律 200 + 业务 code**：

| 状态码 | 使用时机 |
|---|---|
| `200` | 正常返回（含业务失败） |
| `400` | JSON 解析失败、必填 Header 缺失等协议层错误 |
| `401` | 鉴权未通过 |
| `403` | 无权限 |
| `404` | URL 路由不存在 |
| `429` | 触发限流 |
| `500` | 未捕获异常 |

---

## 6. 列表/分页/搜索规范（重点）

### 6.1 设计要点

- **一律 POST，URL 后缀语义化**：`/list`（分页）、`/search`（复杂检索）、`/stat`（聚合统计）、`/export`（导出触发）
- `pagination` 与 `filters` 严格分块，不混层
- 默认页码分页（page + size）；大数据量/深翻页用游标（cursor）
- **page 从 1 开始（1-indexed）**
- 响应头强制 `Cache-Control: no-store`

### 6.2 响应结构

```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "list": [
      { "order_id": "1001", "amount": "99.00", "status": "PAID", "create_time": "2026-04-28T10:15:30+08:00" }
    ],
    "pagination": {
      "page": 1,
      "size": 20,
      "total": 137,
      "total_pages": 7,
      "has_next": true
    },
    "columns": [ ... ]
  },
  "trace_id": "8c41e0a4-77f2-4d88-9a9d-1f1a9c7df222",
  "timestamp": "2026-04-28T10:15:30.123+08:00"
}
```

`data.pagination` 响应字段：

| 字段 | 类型 | 出现于 | 说明 |
|---|---|---|---|
| `page` | int | 页码模式 | 当前页码（回显） |
| `size` | int | 两种模式 | 当前每页条数（可能被裁剪） |
| `total` | int | 页码模式 | 总条数；性能敏感场景可返回 `-1` |
| `total_pages` | int | 页码模式 | 总页数；`total = -1` 时省略 |
| `has_next` | boolean | 两种模式 | 是否还有下一页 |
| `next_cursor` | string | 游标模式 | 下一页游标；无下一页时省略 |

### 6.3 `data.columns` 列定义（★ 各业务接口必须单独说明）

> **当 `pagination.with_column = 1` 时，响应必须附带 `data.columns`；`with_column = 0` 或不传时，禁止返回 `columns`。**

`data.columns` 是**数组**，每个元素代表一列：

```json
"columns": [
  {
    "field": "order_id",
    "label": "订单号",
    "type": "string",
    "sortable": false
  },
  {
    "field": "amount",
    "label": "金额",
    "type": "money",
    "sortable": true,
    "format": "#,##0.00"
  },
  {
    "field": "status",
    "label": "状态",
    "type": "enum",
    "sortable": false,
    "options": [
      { "value": "WAIT_PAY", "label": "待支付" },
      { "value": "PAID",     "label": "已支付" },
      { "value": "CANCELLED","label": "已取消" }
    ]
  },
  {
    "field": "create_time",
    "label": "创建时间",
    "type": "datetime",
    "sortable": true,
    "format": "yyyy-MM-dd HH:mm:ss"
  }
]
```

列元素字段：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `field` | string | **是** | 与 `list` 元素字段名严格一致（snake_case） |
| `label` | string | **是** | 表头显示文案 |
| `type` | string | **是** | `string` / `number` / `money` / `datetime` / `date` / `enum` / `boolean` / `image` / `link` |
| `sortable` | boolean | 否 | 是否支持排序，与 `sort_by` 白名单一致 |
| `options` | array | 否 | 仅 `type = "enum"` 时返回，结构 `[{"value":"PAID","label":"已支付"}]` |
| `format` | string | 否 | 格式提示：如 `"yyyy-MM-dd HH:mm:ss"`、`"#,##0.00"` |
| `visible` | boolean | 否 | 默认是否显示，前端可按用户偏好覆盖 |
| `align` | string | 否 | `left` / `center` / `right`，默认按 type 推断 |
| `width` | int | 否 | 推荐宽度（px），仅作建议 |
| `desc` | string | 否 | 鼠标悬停说明 |

#### ⚠️ 强制要求：每个列表/搜索接口必须在接口文档中明确定义 columns

> 因为每个业务接口的列定义因业务不同而不同（字段名、类型、枚举选项各异），接口文档中**必须显式列出该接口返回的完整 columns 数组**，不得仅写"参见通用结构"。

接口文档中 `columns` 说明模板：

```markdown
#### columns 列定义（with_column=1 时返回）

| field | label | type | sortable | 备注 |
|---|---|---|---|---|
| `order_id` | 订单号 | `string` | false | — |
| `amount` | 金额（CNY） | `money` | true | format: `#,##0.00` |
| `status` | 状态 | `enum` | false | WAIT_PAY/PAID/CANCELLED |
| `create_time` | 创建时间 | `datetime` | true | format: `yyyy-MM-dd HH:mm:ss` |
```

约束：
- `columns` 顺序即前端建议列展示顺序，小版本内保持稳定，新增列追加末尾
- 翻页时 `with_column = 0`，前端缓存首屏列定义
- 含敏感字段的列也必须返回，权限不足走业务码而非删列
- 后端不得通过省略 columns 中的列来表达"无权限"

---

## 7. 命名与字段规范

### URL 命名

- 全小写、短横线分隔：`/pay-channels`，不用 `/pay_channels` 或 `/payChannels`
- 资源名复数：`/orders`、`/users`
- 严禁 query string

### JSON 字段命名

- **统一 snake_case（下划线），严禁驼峰**：`user_id`、`create_time`
- 布尔字段：`is_deleted`、`has_next`、`can_cancel`
- 集合字段：复数，`items`、`tags`
- 不用拼音

### 数据类型

| 数据 | 传输类型 | 格式 |
|---|---|---|
| 时间 | string | **ISO 8601 带时区**，如 `"2026-04-28T10:15:30+08:00"` |
| 时间区间 | string × 2 | `xxx_start` / `xxx_end`，闭区间 |
| 金额 | string | `"99.00"`，字符串防精度丢失 |
| 货币 | string | ISO 4217，`CNY`、`USD` |
| 大整数（雪花 ID） | string | 超 `Number.MAX_SAFE_INTEGER` 必须字符串 |
| 枚举 | string | 全大写下划线：`PAID`、`WAIT_PAY` |
| 布尔 | boolean | `true`/`false`，不用 `"Y"/"N"` 或 `0/1` |
| 百分比 | number | `0~1`，`0.15` 表示 15% |
| 经纬度 | number | 度数十进制，6 位小数 |

### 空值处理

- 响应 `data` 永远是 `{}` 或 `[]`，禁止 `null`
- 字段无值：**省略**该字段，不传 `""` 或 `null`
- 数组字段无值：返回 `[]`，不省略、不为 `null`
- "清空"语义的更新接口：请求 Body 中业务字段可用 `null`，且必须在文档中标注

---

## 8. 错误码规范

### 格式

```
{SERVICE}_{MODULE}_{NNN}
```

| 段 | 规则 | 示例 |
|---|---|---|
| `SERVICE` | 全大写，与数据库 Schema 短名一致 | `TRADE`、`WALLET`、`AUTH`、`COMMON` |
| `MODULE` | 全大写 | `ORDER`、`BALANCE`、`KYC` |
| `NNN` | 固定 3 位数字 | `001`、`201` |

### NNN 区间规则

| 区间 | 类别 | 典型场景 |
|---|---|---|
| `001-099` | 业务参数/校验失败 | 价格超限、参数缺失 |
| `100-199` | 认证/授权/权限 | Token 过期、权限不足 |
| `200-299` | 资源不存在 | 订单/用户不存在 |
| `300-399` | 状态冲突/并发 | 乐观锁冲突、状态机非法流转 |
| `400-499` | 频率/配额/限流 | 提现超日限额 |
| `500-599` | 内部异常/外部依赖 | DB 异常、下游不可用 |

### 通用错误码

| code | HTTP | 含义 |
|---|---|---|
| `0` | 200 | 业务成功 |
| `COMMON_COMMON_001` | 400 | 请求格式错误 |
| `COMMON_COMMON_002` | 200 | 参数校验失败，附 `errors` |
| `AUTH_TOKEN_100` | 401 | 未登录 |
| `AUTH_TOKEN_101` | 401 | 登录已过期 |
| `AUTH_TOKEN_102` | 401 | 凭证无效 |
| `AUTH_PERM_101` | 403 | 无权访问 |
| `COMMON_COMMON_201` | 200 | 资源不存在 |
| `COMMON_COMMON_301` | 200 | 资源状态冲突 |
| `COMMON_COMMON_401` | 429 | 请求过于频繁 |
| `COMMON_COMMON_501` | 500 | 系统繁忙 |
| `COMMON_CRYPTO_001~005` | 400 | 加密信封解析/解密/防重放相关 |

---

## 9. 鉴权方式（⚠️ 方案待定，以下仅供参考）

> 新接入服务暂不依据本章落地；已接入服务保持现状。

- **方式 A**：JWT Bearer Token（内部/Web/App）
- **方式 B**：AppKey + HMAC-SHA256 签名（服务间/开放平台）
- **方式 C**：Session/Cookie（同源 Web 后台）

**Token 严禁出现在 URL 中。**

---

## 10. 应用层加密（敏感接口强制）

以下接口**必须**走混合加密（ECDH/RSA-OAEP + AES-256-GCM）信封：

| 场景 | 示例 |
|---|---|
| 鉴权类 | 登录、注册、改密、Token 刷新 |
| 支付/资金类 | 充提、转账、绑卡 |
| KYC/实名类 | 证件号、人脸、银行卡 |
| 风控敏感数据 | 设备指纹、IP/位置 |

加密接口：`code`/`msg`/`trace_id`/`timestamp` 仍明文，`data` 为加密信封。

---

## 11. 版本控制

- 前端 URL **不含** `/api/v{n}`，版本号由网关拼接
- 同一主版本内只做向后兼容变更（新增字段、新增接口）
- 破坏性变更升主版本：后端并行 v1/v2 → 网关灰度切流 → 稳定后下线旧版
- 快速回滚：网关权重打回旧版本，秒级生效，前端零感知

---

## 12. 完整示例（订单模块）

### 查询订单详情（GET）

```http
GET /order/orders/1001 HTTP/1.1
Authorization: Bearer eyJhbGciOi...
X-Trace-Id: 8c41e0a4-77f2-4d88-9a9d-1f1a9c7df111
```

响应：

```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "order_id": "1001",
    "user_id": "10086",
    "amount": "99.00",
    "currency": "CNY",
    "status": "PAID",
    "create_time": "2026-04-28T10:15:30+08:00"
  },
  "trace_id": "8c41e0a4-77f2-4d88-9a9d-1f1a9c7df111",
  "timestamp": "2026-04-28T10:15:30.123+08:00"
}
```

### 分页查询订单（POST，with_column=1）

```http
POST /order/orders/list HTTP/1.1
Content-Type: application/json
Authorization: Bearer eyJhbGciOi...
X-Trace-Id: 8c41e0a4-77f2-4d88-9a9d-1f1a9c7df222
```

```json
{
  "pagination": {
    "page": 1,
    "size": 20,
    "sort_by": "create_time",
    "order": "desc",
    "with_column": 1
  },
  "filters": {
    "status": ["PAID"]
  }
}
```

响应（`Cache-Control: no-store; X-Robots-Tag: noindex, nofollow`）：

```json
{
  "code": "0",
  "msg": "success",
  "data": {
    "list": [
      { "order_id": "1002", "amount": "120.00", "status": "PAID", "create_time": "2026-04-28T11:00:00+08:00" },
      { "order_id": "1001", "amount": "99.00",  "status": "PAID", "create_time": "2026-04-28T10:15:30+08:00" }
    ],
    "pagination": {
      "page": 1,
      "size": 20,
      "total": 137,
      "total_pages": 7,
      "has_next": true
    },
    "columns": [
      { "field": "order_id",    "label": "订单号",   "type": "string",   "sortable": false },
      { "field": "amount",      "label": "金额",     "type": "money",    "sortable": true, "format": "#,##0.00" },
      { "field": "status",      "label": "状态",     "type": "enum",     "sortable": false,
        "options": [
          { "value": "WAIT_PAY",  "label": "待支付" },
          { "value": "PAID",      "label": "已支付" },
          { "value": "CANCELLED", "label": "已取消" }
        ]
      },
      { "field": "create_time", "label": "创建时间", "type": "datetime", "sortable": true, "format": "yyyy-MM-dd HH:mm:ss" }
    ]
  },
  "trace_id": "8c41e0a4-77f2-4d88-9a9d-1f1a9c7df222",
  "timestamp": "2026-04-28T10:15:30.123+08:00"
}
```

> 翻页时 `with_column = 0`，前端缓存首屏 `columns`。

### 创建订单（POST）

```json
{
  "idempotency_key": "create-8c41e0a4-77f2-4d88-9a9d-1f1a9c7df333",
  "user_id": "10086",
  "items": [
    { "sku_id": "S-001", "quantity": 2 }
  ]
}
```

响应：

```json
{
  "code": "0",
  "msg": "success",
  "data": { "order_id": "1003", "status": "WAIT_PAY" },
  "trace_id": "...",
  "timestamp": "2026-04-28T10:15:30.123+08:00"
}
```

---

## 13. 接入评审清单

- [ ] URL 中无 `?` 与 query string，动态参数全走 POST Body
- [ ] GET 仅用于"按路径参数取单资源"，无 Body、无 query
- [ ] 前端可见 URL 不含 `api`、`v1`、`v2`，版本号由网关拼接
- [ ] 列表/分页/搜索使用 `POST /resource/list`（或 `/search`/`/stat`），`pagination` 与 `filters` 分块
- [ ] 响应使用 `data.list` + `data.pagination` 双块结构
- [ ] `filters` 内无保留字段名：`page`/`size`/`sort_by`/`order`/`cursor`/`with_column`
- [ ] 列表接口支持 `pagination.with_column`：`1` 时返回 `data.columns`（数组），`0` 时不返回
- [ ] **接口文档中显式列出该接口的 columns 定义表格**（field / label / type / sortable / 备注）
- [ ] `page` 从 `1` 开始（1-indexed），服务端已对缺失/≤0/非整数做规范化
- [ ] 响应外壳 `data` 永远是 `{}` 或 `[]`，禁止 `null`
- [ ] `trace_id` 与请求头 `X-Trace-Id` 一致并原样回传
- [ ] 创建/写接口的幂等键命名为 `idempotency_key`，不与 `trace_id` 复用
- [ ] 业务错误返回 HTTP 200 + `{SERVICE}_{MODULE}_{NNN}` 三段业务码，NNN 落在正确区间
- [ ] **所有 JSON 字段统一 snake_case，禁止驼峰**；时间 ISO 8601，金额字符串，枚举大写下划线
- [ ] 响应外壳 `timestamp` 是 ISO 8601 带时区字符串
- [ ] 列表/敏感接口响应头：`Cache-Control: no-store`、`X-Robots-Tag: noindex, nofollow`
- [ ] 涉及鉴权/资金/KYC/风控的接口按 §10 走加密信封

---

## 附录 A：字段命名速查

| 含义 | 字段名 | 位置 |
|---|---|---|
| 主键 ID | `id` / `order_id` | list 元素 |
| 创建时间 | `create_time` | list 元素 |
| 更新时间 | `update_time` | list 元素 |
| 状态 | `status` | list 元素 |
| 金额 | `amount`（字符串） | list 元素 |
| 列表 | `list` | `data` 下 |
| 分页结构体 | `pagination` | 请求 Body / `data` 下 |
| 总数 | `total` | `data.pagination` |
| 总页数 | `total_pages` | `data.pagination` |
| 是否还有下一页 | `has_next` | `data.pagination` |
| 下一页游标 | `next_cursor` | `data.pagination` |
| 排序字段 | `sort_by` | `pagination` |
| 排序方向 | `order` | `pagination` |
| 是否返回表头 | `with_column` | `pagination` |
| 列定义数组 | `columns` | `data` 下，仅 `with_column=1` 时 |
| 列字段 key | `field` | `columns` 元素 |
| 列显示标题 | `label` | `columns` 元素 |
| 列数据类型 | `type` | `columns` 元素 |
| 过滤条件容器 | `filters` | 请求 Body |
| 关键字搜索 | `keyword` | `filters` 内 |
| 幂等键 | `idempotency_key` | 请求 Body 根节点 |

---

## 附录 B：常见反例

| ❌ 反例 | ✅ 正确做法 |
|---|---|
| `GET /orders?page=1&size=20` | `POST /orders/list`，Body 携带分页 |
| `GET /users?phone=13800000000` | `POST /users/search`，敏感字段进 Body |
| `{ "page": 1, "size": 20, "status": ["PAID"] }` 混层 | `{ "pagination": {...}, "filters": { "status": [...] } }` |
| `data: null` | `data: {}` 或 `data: []` |
| 列表为空返回 `null` | `"list": []` |
| `timestamp: 1745798400000`（毫秒数字） | `"2026-04-28T10:15:30.123+08:00"` |
| 金额 `99.0`（number） | `"99.00"`（string） |
| 枚举 `status: 1` | `"PAID"` |
| 用 `trace_id` 同时做幂等键 | 幂等用 `idempotency_key`，追踪用 `trace_id` |
| 翻页每次都返回 `columns` | 翻页 `with_column=0`，前端缓存首屏列定义 |
| 通过省略 columns 中的列表达"无权限" | 权限不足走业务码，columns 保持稳定 |
| 接口文档写"columns 参见通用结构" | 接口文档必须显式列出该接口的完整列定义表格 |
| `page` 从 `0` 开始（Spring Data 默认） | 1-indexed，Spring 项目配 `one-indexed-parameters=true` |
| 前端写 `/order/api/v1/orders/list` | `fetch('/order/orders/list')`，版本号网关拼 |
| 字段名 `orderId`、`createTime`（驼峰） | `order_id`、`create_time`（snake_case） |
| `traceId`、`totalPages`、`hasNext` | `trace_id`、`total_pages`、`has_next` |
| `idempotencyKey` | `idempotency_key` |
