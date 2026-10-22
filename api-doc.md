---
title: "HTTP 接口规范与接口文档模板"
version: "2.0.0-draft"
last_modified: "2026-10-22"
---
<!-- [skill: go-team-standards · 接口规范 · 接口文档] REST + JSON 统一响应体接口规范（Gin） -->

# HTTP 接口规范与接口文档模板

> 版本：V2.0.0-draft | 状态：草案 | 适用：Go + Gin 后端对外 / 对前端暴露的 HTTP 接口
>
> 风格：**RESTful + JSON + 统一响应体**。响应外壳、错误码、分页的结构与公共库 `gin-go-common-lib/response`、`errors` 保持一致（以公共库代码为准）。错误码体系见 [error-codes.md](error-codes.md)，前端对接约定见 [frontend.md](frontend.md)。
>
> **与 v1.x 的差异**：v1.x 只用 GET / POST、参数全放 Body、禁用 query string、业务错误一律 HTTP 200。v2.0 改为标准 REST 方法，GET 用 query string，**HTTP 状态码按错误类别如实返回**。

---

## 1. 核心原则

1. **资源导向**：URL 表达资源，HTTP 方法表达动作
2. **统一外壳**：所有响应（成功、失败）都是同一个 `Envelope`，前端只写一套解析
3. **HTTP 状态码如实**：参数错 400、未登录 401、无权限 403、不存在 404、冲突 409、限流 429、内部错 500；业务细分靠 `code`
4. **契约先行**：先写接口文档并评审，再写代码；字段以文档为准，不得自行发明
5. **向后兼容**：同一主版本内只增不改不删

---

## 2. URL 与方法

### 2.1 URL 格式

```
/api/v{major}/{resource}[/{id}][/{sub-resource}][/{action}]
```

| 部分 | 规则 | 示例 |
|---|---|---|
| `v{major}` | 主版本号，破坏性变更才升 | `v1` |
| `resource` | **复数名词**、小写、短横线分隔 | `orders`、`pay-channels` |
| `id` | 路径参数，UUID 36 位字符串 | `0194f3a2-7c1e-7b52-9d0a-3f6c2e8a1b45` |
| `action` | 非 CRUD 的状态流转，动词，小写短横线 | `cancel`、`batch-export` |

**禁止：** 动词做资源名（`/getOrders`、`/createOrder`）；大写、下划线、驼峰；路径里放敏感信息（手机号、证件号、Token）；路径里体现业务主域名（`/earn/orders`，主域由服务划分承载）。

### 2.2 HTTP 方法

| 方法 | 用途 | 幂等 | 示例 |
|---|---|---|---|
| `GET` | 查询单个 / 列表，**无副作用** | 是 | `GET /api/v1/orders/{id}`、`GET /api/v1/orders?page=1&page_size=20` |
| `POST` | 创建；非 CRUD 的状态流转 | 否（须带幂等键） | `POST /api/v1/orders`、`POST /api/v1/orders/{id}/cancel` |
| `PUT` | 整体替换 | 是 | `PUT /api/v1/users/me/settings` |
| `PATCH` | 局部更新 | 视实现 | `PATCH /api/v1/orders/{id}` |
| `DELETE` | 删除（本项目允许物理删除，见 [database.md](database.md) §4.4） | 是 | `DELETE /api/v1/orders/{id}` |

* 复杂搜索（条件很多、含数组 / 嵌套）可用 `POST /api/v1/orders/search`，请求体放条件；这是唯一允许的“用 POST 查询”
* 资金类写接口（下单、提现、划转）一律 `POST` + 幂等键，**不提供 `PUT` / `DELETE` 变体**
* GET 响应对敏感数据加 `Cache-Control: no-store`

---

## 3. 请求规范

### 3.1 请求头

| Header | 必填 | 说明 |
|---|---|---|
| `Content-Type` | 有 Body 时 | `application/json; charset=utf-8` |
| `Authorization` | 需鉴权的接口 | `Bearer <JWT>`；**Token 严禁出现在 URL 中** |
| `X-Trace-Id` | 建议 | 全链路追踪 ID；不传则服务端生成；响应头与响应体 `trace_id` 原样回传 |
| `X-Idempotency-Key` | 创建 / 资金类写接口 | 幂等键，客户端生成（UUID），见 §3.4 |
| `X-User-Id` | — | **仅由网关在鉴权通过后注入**，供内部服务读取。对外请求中该头必须被网关丢弃并覆盖，服务**不得**信任来自公网客户端的该头 |

### 3.2 Query 参数（GET）

* 命名 `snake_case`
* 分页 / 排序保留参数（公共库 `response.PaginationIn`）：

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `page` | int | `1` | 页码，**从 1 开始**，与 `cursor` 互斥 |
| `page_size` | int | `20` | 每页条数，最大 `100`，超出服务端裁剪 |
| `cursor` | string | — | 游标分页：上次响应的 `next_cursor`，首次不传 |
| `sort_by` | string | 业务定 | 排序字段，服务端**白名单**校验 |
| `order` | string | `desc` | `asc` / `desc` |

* 过滤条件用平铺参数：`status=PAID&create_time_start=1761109000000&create_time_end=...`
* 数组参数重复传参：`status=PAID&status=SHIPPED`
* 过滤参数名不得与上面的保留参数重名
* 服务端用 `response.BindQueryOrFail` 绑定与校验

### 3.3 请求体（POST / PUT / PATCH）

* 根节点必须是对象 `{}`
* 字段 `snake_case`，**禁止驼峰**
* 可选字段缺失时省略，不传 `null` 占位；“清空”语义的更新接口除外，须在文档中标注
* 请求体 ≤ 2 MB，更大走文件上传
* 服务端用 `response.BindJSONOrFail` 绑定与校验（校验标签用 `validator`）

### 3.4 幂等性

* 创建类、资金类写接口必须支持幂等：客户端传 `X-Idempotency-Key`，24 小时内同一 key 的重复请求不得重复执行
* 公共库 `idempotency.Middleware` 对重复请求返回 `HTTP 409` + `COMMON_COMMON_301`；该中间件在 Redis 异常时**放行**，因此**最终幂等必须由 MySQL 唯一键保证**（见 [redis.md](redis.md) §6）
* **`X-Idempotency-Key` 与 `X-Trace-Id` 是两个概念，绝不复用**
* 状态流转类：目标状态与当前状态相同时视为幂等成功

---

## 4. 响应规范

### 4.1 统一外壳（强制）

```json
{
  "code": "0",
  "msg": "ok",
  "data": {},
  "trace_id": "8c41e0a4-77f2-4d88-9a9d-1f1a9c7df222",
  "timestamp": "2026-10-22T08:15:30Z"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | string | `"0"` 成功；其他为业务错误码（`{SERVICE}_{MODULE}_{NNN}`） |
| `msg` | string | 成功为 `ok`；失败为可直接展示给用户的提示 |
| `data` | object / array | 业务数据 |
| `trace_id` | string | 与 `X-Trace-Id` 一致 |
| `timestamp` | string | 响应时间，**ISO 8601 UTC**（`2026-10-22T08:15:30Z`） |

后端一律通过公共库输出，**禁止直接 `c.JSON()`**：`response.JSON` / `JSONList` / `Fail` / `FailErr` / `FailValidation`（公共库会统一把 key 转为 `snake_case`）。

### 4.2 `data` 取值约定

| 接口类型 | `data` 内容 |
|---|---|
| 列表 / 分页 | `{ "list": [], "pagination": {...} }`；无数据时 `list: []`，**不是 `null`** |
| 单资源详情 | 资源对象；不存在返回 `404` + `COMMON_COMMON_201`（或业务码），`data` 为 `null` 或 `{}` |
| 创建 / 状态流转 | 至少返回受影响主键，如 `{ "order_id": "..." }` |
| 删除 | `{}` |
| 批量操作 | `{ "success_ids": [], "failures": [] }`，空时也用 `[]` |
| 校验失败 | `{ "errors": [{ "field": "price", "message": "price must be greater than 0" }] }` |

### 4.3 列表与分页

```json
{
  "code": "0",
  "msg": "ok",
  "data": {
    "list": [
      { "order_id": "0194f3a2-...", "order_amount": "99.00", "order_status": "PAID", "create_time": 1761109000123 }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 137,
      "has_more": true,
      "next_cursor": ""
    },
    "warnings": []
  },
  "trace_id": "...",
  "timestamp": "..."
}
```

| 字段 | 出现于 | 说明 |
|---|---|---|
| `page` / `page_size` | 页码模式 | 回显（`page_size` 可能被裁剪） |
| `total` | 页码模式 | 总条数；**游标模式固定 `-1`**（不做 count） |
| `has_more` | 两种模式 | 是否还有下一页 |
| `next_cursor` | 游标模式 | 下一页游标；最后一页为空 |
| `warnings` | 可选 | 查询降级、数据延迟等提示 |

* **默认页码分页；数据量大（单表 > 100 万）或深翻页必须用游标分页**
* `page * page_size > 10000` 时服务端拒绝并提示改用 `cursor`（返回 `COMMON_COMMON_002`）
* 游标内容对前端是**不透明字符串**，前端不得解析或自行构造
* MySQL 侧游标写法见 [database.md](database.md) §7.3

### 4.4 HTTP 状态码（与公共库映射一致）

| HTTP | 业务码示例 | 场景 |
|---|---|---|
| 200 | `0` | 成功 |
| 400 | `COMMON_COMMON_001` / `002` | JSON 语法错误、类型不匹配 / 字段校验失败 |
| 401 | `AUTH_TOKEN_100` / `101` / `102` | 未登录 / 已过期 / 无效 |
| 403 | `AUTH_PERM_101` | 无权限 |
| 404 | `COMMON_COMMON_201` | 资源不存在 |
| 409 | `COMMON_COMMON_301` | 状态冲突、乐观锁冲突、重复请求 |
| 429 | `COMMON_COMMON_401` | 限流 |
| 500 | `COMMON_COMMON_501` | 内部异常 |
| 502 | `COMMON_COMMON_502` | 上游依赖异常 |

* 业务码未配置映射时，`response.Fail` 默认 400
* **前端以 `code` 判断业务结果，不要只看 HTTP 状态码**

---

## 5. 命名与数据类型

### 5.1 命名

* URL：小写短横线，资源名复数
* JSON 字段：**统一 `snake_case`**，与数据库字段命名一致（见 [field-naming.md](field-naming.md)）：`uid`、`platform_id`、`order_id`
* 布尔：`is_` / `has_` / `can_` 前缀
* 集合字段用复数；不用拼音

### 5.2 数据类型

| 数据 | JSON 类型 | 格式 |
|---|---|---|
| 业务时间点 | number | **毫秒时间戳（int64，UTC）**，字段名以 `_time` 结尾：`create_time`、`update_time`、`pay_time` |
| 响应外壳时间 | string | `timestamp` 为 ISO 8601 UTC |
| 金额 / 数量 / 价格 | **string** | `"99.00"`、`"0.00012345"`；**禁止 number**，防精度丢失 |
| 主键 / 业务 ID | **string** | UUID 36 位小写（`BINARY(16)` 在序列化层转换，见 [database.md](database.md) §4.3） |
| 枚举 | string | 大写下划线：`PAID`、`WAIT_PAY`；后端在 biz 层与数据库的 `TINYINT` 互转 |
| 布尔 | boolean | `true` / `false`；不用 `0/1`、`"Y"/"N"` |
| 比率 | string | `"0.15"` 表示 15% |
| 交易对 | string | `BASE-QUOTE`，如 `BTC-USDT`（见 [field-naming.md](field-naming.md) §1.4） |

### 5.3 空值

* 列表字段无数据返回 `[]`，不省略、不为 `null`
* 单值字段无值：**省略**，不传 `""` 或 `null`
* 响应的 `data` 在成功时不得为 `null`

---

## 6. 鉴权与安全

* 默认 **JWT Bearer**（HS256，Access Token 15 分钟）；刷新、登出、多端互踢等细节由 `auth` 服务的设计文档定义
* 服务间调用走内网并带内部凭证，不复用用户 Token
* **敏感接口**（登录、改密、提现、绑卡、KYC）须 HTTPS，且响应头 `Cache-Control: no-store`；如业务要求应用层加密，信封格式在对应接口文档中单独定义
* 限流：网关层统一限流；登录、验证码、提现等接口须有**更严**的业务限流，返回 `429` + `COMMON_COMMON_401`
* 所有写接口必须记录审计日志（操作人 `uid`、`trace_id`、`action`、目标对象），写入 `log_*` 表（见 [database.md](database.md) §4.9）
* 日志中**禁止**出现 Token、密码、完整证件号 / 卡号

---

## 7. 版本控制

* URL 带主版本：`/api/v1/...`
* 同一主版本内仅做向后兼容变更：新增接口、新增**可选**字段
* 破坏性变更（删字段、改类型、改语义）升主版本，并行运行 v1 / v2，稳定后下线旧版并提前公告
* 响应中新增字段，前端必须容忍未知字段

---

## 8. 接口文档模板

每个接口按此模板写在对应服务的 `docs/api/` 下，评审通过后再开发。

````markdown
### {接口名称}

- **路径**：`POST /api/v1/orders`
- **鉴权**：需要登录（Bearer）
- **幂等**：必须携带 `X-Idempotency-Key`
- **限流**：每用户 10 次/秒
- **负责人**：{服务 owner}

#### 请求头
（只列本接口特有的，通用头见接口规范）

#### Query 参数 / 请求体

| 字段 | 类型 | 必填 | 说明 | 校验 |
|---|---|---|---|---|
| `symbol` | string | 是 | 交易对，`BASE-QUOTE` | 必须在上架交易对内 |
| `order_side` | string | 是 | `BUY` / `SELL` | 枚举 |
| `price` | string | 是 | 委托价 | > 0，精度 ≤ 交易对价格精度 |
| `quantity` | string | 是 | 数量 | ≥ 最小下单量 |

#### 成功响应 `data`

| 字段 | 类型 | 说明 |
|---|---|---|
| `order_id` | string | 订单 ID（UUID） |

#### 错误码

| code | HTTP | 触发条件 |
|---|---|---|
| `COMMON_COMMON_002` | 400 | 参数校验失败 |
| `TRADE_ORDER_001` | 400 | 价格超出允许区间 |
| `WALLET_BALANCE_001` | 400 | 余额不足 |
| `COMMON_COMMON_301` | 409 | 重复请求 |

#### 示例
（请求 / 响应各一份完整 JSON）

#### 变更记录
````

**列表接口额外必须写明：** 支持的过滤参数、`sort_by` 白名单与默认排序、默认 / 最大 `page_size`、是否支持游标、`list` 元素字段表。

---

## 9. 完整示例（订单）

### 9.1 查询订单详情

```http
GET /api/v1/orders/0194f3a2-7c1e-7b52-9d0a-3f6c2e8a1b45 HTTP/1.1
Authorization: Bearer <jwt>
X-Trace-Id: 8c41e0a4-77f2-4d88-9a9d-1f1a9c7df222
```

```json
{
  "code": "0",
  "msg": "ok",
  "data": {
    "order_id": "0194f3a2-7c1e-7b52-9d0a-3f6c2e8a1b45",
    "symbol": "BTC-USDT",
    "order_side": "BUY",
    "order_type": "LIMIT",
    "price": "65000.00",
    "quantity": "0.5",
    "filled_qty": "0.2",
    "order_status": "PARTIALLY_FILLED",
    "create_time": 1761109000123,
    "update_time": 1761109005456
  },
  "trace_id": "8c41e0a4-77f2-4d88-9a9d-1f1a9c7df222",
  "timestamp": "2026-10-22T08:15:30Z"
}
```

### 9.2 分页查询订单

```http
GET /api/v1/orders?symbol=BTC-USDT&order_status=PAID&page=1&page_size=20&sort_by=create_time&order=desc HTTP/1.1
```

响应见 §4.3。

### 9.3 创建订单

```http
POST /api/v1/orders HTTP/1.1
Content-Type: application/json; charset=utf-8
Authorization: Bearer <jwt>
X-Idempotency-Key: 7f8d3c5e-2b1a-4e6f-9c0d-8a7b6c5d4e3f

{
  "symbol": "BTC-USDT",
  "order_side": "BUY",
  "order_type": "LIMIT",
  "price": "65000.00",
  "quantity": "0.5"
}
```

成功：

```json
{ "code": "0", "msg": "ok", "data": { "order_id": "0194f3a2-7c1e-7b52-9d0a-3f6c2e8a1b45" }, "trace_id": "...", "timestamp": "..." }
```

校验失败（HTTP 400）：

```json
{
  "code": "COMMON_COMMON_002",
  "msg": "validation failed",
  "data": { "errors": [ { "field": "price", "message": "price must be greater than 0" } ] },
  "trace_id": "...",
  "timestamp": "..."
}
```

> 公共库 `response.BindJSONOrFail` 使用 Gin 默认校验器（`binding` 标签），错误里的 `field` 是 Go 结构体字段名（`Price`），**不是 JSON 字段名**。公共库另有 `validation.ValidateAndFail`（`validate` 标签），会改用 JSON 字段名（`price`）。两者标签不同、不要混用；要让前端拿到 JSON 名，用后者（见 §11 待确认）。

### 9.4 Gin 实现骨架

```go
// handler：只做参数绑定、调用 biz、输出响应，不含业务逻辑
func (h *OrderHandler) Create(c *gin.Context) {
	var req CreateOrderReq
	if !response.BindJSONOrFail(c, &req) {
		return
	}
	uid, ok := header.GetUserID(c.Request.Context())
	if !ok {
		response.FailErr(c, errors.ErrUnauthorized("missing user"))
		return
	}
	orderID, err := h.uc.CreateOrder(c.Request.Context(), uid, req.ToDomain())
	if err != nil {
		response.FailErr(c, err) // *errors.AppError 自动映射 HTTP 状态码与业务码
		return
	}
	response.JSON(c, gin.H{"order_id": orderID})
}
```

---

## 10. 接入评审清单

- [ ] URL：资源复数、小写短横线、带 `/api/v{n}`，无动词接口名
- [ ] 方法使用正确；GET 无副作用；资金类写接口只有 POST + 幂等键
- [ ] 请求 / 响应字段 `snake_case`，与数据库命名一致
- [ ] 金额、数量、价格、ID 均为 string；业务时间为毫秒 int64
- [ ] 响应为统一外壳，经公共库 `response` 输出
- [ ] 错误码已按 [error-codes.md](error-codes.md) 定义，HTTP 状态码映射正确
- [ ] 列表接口有 `sort_by` 白名单、`page_size` 上限、大数据量支持游标
- [ ] 写接口有幂等设计，且 MySQL 有唯一键兜底
- [ ] 敏感接口：HTTPS、`no-store`、限流、审计日志
- [ ] 日志无敏感信息
- [ ] 接口文档已按模板写完并评审通过

---

## 11. 待确认

| # | 问题 | 当前处理 |
|---|---|---|
| 1 | `BindJSONOrFail` 的校验错误 `field` 为 Go 字段名；`validation.ValidateAndFail` 则为 JSON 名 | 文档已标注；建议统一为 JSON 名（给 `BindJSONOrFail` 的校验器注册 JSON tag 名，或统一改用 `validation` 包） |
| 2 | 公共库 `idempotency.Middleware` 重复请求返回 409 且不回放首次结果 | 规范要求“返回首次结果”较难实现；当前按“拒绝重复”执行，若需回放须在业务层存首次响应 |
| 3 | 鉴权细节（刷新、多端、Token 撤销） | 默认 JWT Bearer，细节待 `auth` 服务方案 |
| 4 | 网关是否统一加 `/api/v{n}` 前缀 | 本文要求后端自己的路由带 `/api/v1` |

---

## 附录：常见反例

| ❌ 反例 | ✅ 正确 |
|---|---|
| `POST /getOrderList` | `GET /api/v1/orders` |
| `GET /api/v1/orders?pageNo=1&pageSize=20` | `GET /api/v1/orders?page=1&page_size=20` |
| `{ "price": 65000.5 }` | `{ "price": "65000.5" }` |
| `{ "orderId": "..." }` | `{ "order_id": "..." }` |
| 业务失败返回 HTTP 200 + `code: "0"` 以外却不带语义状态码 | 按 §4.4 返回对应 HTTP 状态码 |
| `c.JSON(200, gin.H{...})` | `response.JSON(c, data)` |
| URL 带 Token：`?token=xxx` | `Authorization: Bearer ...` |
| `data: null`（成功时） | `data: {}` 或 `[]` |
| 前端自己拼游标 | 原样回传 `next_cursor` |
