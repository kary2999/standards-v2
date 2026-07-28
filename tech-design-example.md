---
title: "后端技术方案 - 示例用户中台"
version: "1.0.0"
last_modified: "2026-04-29"
source: "技术规范.2026.04.29 / 后端技术方案 - 示例用户中台.md (示例样本)"
---

# 后端技术方案 - 示例用户中台

# 用户中台技术方案

## 目录


 1. 背景与概述
 2. 目标与原则
 3. 规范定义
 4. 核心业务流程
 5. 应用架构
 6. 系统架构
 7. 功能模块设计
 8. 接口清单
 9. 数据库设计
10. 边界与异常处理


---

## 1. 背景与概述

### 1.1 项目背景

公司内部系统数量持续增长，各系统独立维护用户体系，导致账号管理分散、离职账号无法统一失效、权限配置无法跨系统管理等问题。用户中台作为统一 IAM（Identity and Access Management）平台，解决：

* 员工账号分散于各系统，入离职操作繁琐且存在遗漏风险
* 各系统独自实现 RBAC，逻辑不统一，无法跨系统配置权限
* 无统一审计日志，操作行为无法全链路追溯

### 1.2 范围说明

| 类别  | 在范围内 | 不在范围内 |
|-----|------|-------|
| 身份认证 | Google Workspace SSO、AT/RT 颁发、MFA | C 端用户注册登录（2期） |
| 权限管控 | 菜单级 RBAC、角色管理、权限双向同步 | 按钮级、数据级权限（2期） |
| 系统接入 | 系统注册、common-lib SDK（Go + TS）、Kafka 双向同步 | 各系统内部业务逻辑 |
| 审计  | 用户中台自身操作审计日志 | 子系统业务操作日志 |
| 平台管理 | —    | 联盟站生命周期管理（2期） |


---

## 2. 目标与原则

### 2.1 业务目标

* 统一入口：所有内部员工通过 Google SSO 单点登录所有内部系统
* 即时生效：权限变更后 ≤ 15 分钟所有系统生效，Kafka 事件秒级通知
* 前后端一致：前端 RouteGuard 与后端 AuthMiddleware 基于同一套 menu_key，权限状态同步
* 强制标准：所有系统通过 common-lib 接入，禁止自建 RBAC 逻辑
* 可审计：账号和权限变更全链路记录，保留 2 年

### 2.2 技术目标

| 指标  | 目标值 |
|-----|-----|
| 登录流程 P99 延迟 | < 500ms（含 Google SSO 处理） |
| AT 本地验签延迟 | < 1ms |
| 权限查询延迟（L1/L2） | < 1ms |
| 权限变更生效时间 | ≤ 15 分钟（AT 黑名单）+ Kafka 秒级 L1 失效 |
| 系统可用性 | ≥ 99.9% |
| 审计日志保留 | 2 年 |

### 2.3 设计原则


1. 最小权限：账号默认无任何系统访问权限，需显式分配角色
2. 强制标准化：RBAC 实现由 common-lib IAM 模块内置，子系统无选择权
3. 前后端同源：前后端基于同一份 menu_key 声明，前端路由守卫与后端接口拦截使用相同的权限粒度
4. 读写分离：共享 Redis 权限缓存只由 iam-center-core-service 写入，子系统只读
5. 可用性优先：权限查询不依赖 iam-center-core-service 实时可用，走共享 Redis 三级缓存
6. 主AT与权限包分离：AT 只携带身份信息，权限包独立存于 Redis，降低 token 体积


---

## 3. 规范定义

### 3.1 术语表

| 术语  | 定义  |
|-----|-----|
| 主AT | Access Token 精简版，只含身份信息，不含菜单权限明细 |
| 权限包 | 存储于共享 Redis 的菜单权限列表，与主AT分离 |
| system_id | 子系统全局唯一标识，在配置文件中声明，启动时自动注册 |
| menu_key | 菜单权限项全局唯一键，格式：{system_id}:{模块}:{页面}，前后端共享同一定义 |
| checksum | 菜单列表 MD5 摘要，上行同步时用于判断是否需要 diff |
| L1 缓存 | 子系统进程内存缓存（sync.Map），TTL 30 秒 |
| L2 缓存 | 用户中台共享 Redis，子系统只读，TTL 5 分钟 |
| L3 存储 | 用户中台 PostgreSQL 主库，权限数据源 |
| 上行同步 | 子系统 → 用户中台，菜单变更通知 |
| 下行同步 | 用户中台 → 子系统，权限变更推送 |

### 3.2 编码规范

遵循团队《Go 编码风格规范》与《命名与日志规范手册》，关键约束：

* 项目命名：四段式 \[业务域\]-\[子系统\]-\[功能/模块\]-\[类型\]，类型后缀区分 -service / -lib / -job / -web
* 项目结构：统一采用 cmd/、internal/（handler → service → repository 分层）、api/ 脚手架目录结构
* 日志：统一使用 slog（Go 1.21+），JSON 格式，必须携带 trace_id + span_id（W3C Trace Context 标准）
* 接口定义：接口放使用方包内；单方法接口用 er 后缀，多方法用名词
* 枚举：用 type XxxStatus int + iota 定义，禁止裸 int 常量
* 服务间通信：gRPC（内部）+ REST（管理后台），proto 文件由 Buf 管理
* 时间格式：ISO 8601，统一 UTC 时区
* 请求 ID：所有请求携带 X-Request-ID（UUID v4），从 OTel context 传播

### 3.3 账号状态枚举

```go
type AccountStatus int

const (
    AccountStatusPending  AccountStatus = iota + 1 // 待激活
    AccountStatusActive                             // 正常
    AccountStatusDisabled                           // 已禁用
)
```

### 3.4 菜单状态枚举

```go
type MenuStatus int

const (
    MenuStatusActive     MenuStatus = iota + 1 // 正常
    MenuStatusDeprecated                        // 子系统已删除但有角色引用
    MenuStatusDeleted                           // 已完全删除
)
```

### 3.5 Redis Key 规范

遵循团队规范格式：\[业务名\]:\[模块\]:\[唯一标识\]

| Key 模式 | 说明  | TTL | 写入方 |
|--------|-----|-----|-----|
| iam:perms:{uid}:{system_id} | 用户在某系统的菜单权限包 | 5min（变更时主动 DEL） | iam-center-core-service |
| iam:token:revoked:{jti} | AT 黑名单 | AT 剩余有效期 | iam-center-core-service |
| iam:token:rt:{uid}:{device_id} | Refresh Token | 30d | iam-center-core-service |
| iam:session:info:{uid} | 用户 session 信息 | 30d | iam-center-core-service |

### 3.6 Kafka Topic 规范

遵循团队规范格式：\[环境\]\[服务名\]\[业务语义\]_\[动作\]

| Topic | 方向  | 生产者 | 消费者 | Payload 摘要 |
|-------|-----|-----|-----|------------|
| {env}_iam_menus_synced | 上行  | 子系统（启动时） | iam-center-core-service | {system_id, checksum, menus\[\]} |
| {env}_iam_permission_changed | 下行  | iam-center-core-service | 对应子系统 | {uid, system_id, menus\[\]} |
| {env}_iam_user_suspended | 下行  | iam-center-core-service | 所有子系统 | {uid}      |
| {env}_iam_roles_changed | 下行  | iam-center-core-service | 对应子系统 | {uid, system_id, added\[\], removed\[\]} |
| {env}_iam_menu_deprecated | 内部  | iam-center-core-service | iam-center-core-service（告警） | {system_id, menu_key, role_ids\[\]} |

{env} 取值：prod / staging / dev，由 IDP 在部署时注入，代码中不硬编码环境名。


---

## 4. 核心业务流程

### 4.1 系统注册与菜单上行同步

触发时机： 子系统每次启动时，由 common-lib/iam（Go）自动执行。

```
子系统启动（common-lib/iam bootstrap hook）
  → 读取 config.yaml: system_id, display_name, menus[]
  → 计算 menus[] 的 MD5 checksum
  → 发 Kafka: {env}_iam_menus_synced {system_id, checksum, menus[]}

iam-center-core-service 消费 {env}_iam_menus_synced
  → 对比 iam.systems.checksum
  → 一致 → 忽略
  → 不一致 → 执行 diff：
      新增 menu_key → INSERT iam.system_menus
      name/path/sort 变更 → UPDATE iam.system_menus
      key 不在列表中（删除）→
          查 iam.role_menus 是否有引用
          无引用 → UPDATE status = deleted
          有引用 → UPDATE status = deprecated
                   发 Kafka: {env}_iam_menu_deprecated 告警
  → UPDATE iam.systems SET checksum = ?, last_sync_at = NOW()
```

### 4.2 登录与 AT 颁发（主AT + 权限包分离）

核心原则：

* 登录相关接口（`/api/v1/auth/*`）在网关配置为白名单透传，网关不做任何 JWT 验签，直接转发至 iam-center-core-service 处理
* 所有其他业务接口，前端必须携带 IAM 颁发的 AT，由网关统一做 JWT 验签 + 权限检查

```
【登录流程】

前端检测未登录
  → 跳转至网关登录入口：GET /api/v1/auth/google?system_id=xxx&redirect_uri=xxx
  → 网关透传至 iam-center-core-service（/api/v1/auth/google 在网关白名单中）
  → iam-center-core-service 生成 state，重定向至 Google OAuth 授权页

Google OAuth 回调：GET /api/v1/auth/google/callback
  → 网关透传至 iam-center-core-service（同为白名单接口）
  → iam-center-core-service 用 code 换取 Google id_token
  → 验证 id_token（RS256、aud、iss、exp、nonce）
  → 查 iam.accounts（google_sub 匹配）
  → 不存在 / disabled → 403
  → pending → 激活账号

权限查询（颁发前）
  → GET Redis: iam:perms:{uid}:{system_id}
  → Miss → 查 PostgreSQL 计算权限并集 → 写入 Redis

颁发主AT（RS256，15min）
  payload: {uid, username, user_type, platform_id, system_id, email, roles[], jti, iat, exp}
  不含 menus[]（权限包独立存 Redis）

写入 RT（Redis: iam:token:rt:{uid}:{device_id}，TTL 30d）
iam-center-core-service 将主AT + RT 返回前端

【业务请求流程】

前端携带 AT → Authorization: Bearer <token>
  → 网关 JWT 验签（IAM SDK AuthMiddleware）
  → 黑名单检查 → 封禁检查 → 权限加载
  → 注入身份 Header（X-User-Id / X-Username / X-Role / X-System-Id）
  → 透传至下游业务微服务（下游无需重复验签）

common-lib/iam（Go）AuthMiddleware（各业务服务可选，部署在网关后则直接读 Header）
  → L0 验签 → L1 查 menus[] → L1 Miss → L2 回填 → 注入 ctx.menus

common-lib/iam（TS）
  → iamClient.init(accessToken) → 初始化前端 menus[]
  → RouteGuard / MenuGuard 开始工作
```

### 4.3 权限下行同步（即时生效）

```
管理员修改角色菜单权限 或 调整账号角色
  → iam-center-core-service 写入 PostgreSQL
  → DEL Redis: iam:perms:{uid}:{system_id}
  → SET Redis: iam:token:revoked:{jti}（AT 黑名单）
  → 发 Kafka: {env}_iam_permission_changed {uid, system_id, menus[]}1234

后端 common-lib/iam（Go）消费
  → 清除 L1 内存缓存 {uid}:{system_id}

前端响应
  → 下次请求后端 → AT 黑名单命中 → 401
  → common-lib/iam（TS）onUnauthorized 触发
  → 重新登录 → 获得新 AT → iamClient.init() → 菜单重新渲染
```

### 4.4 账号禁用

```
管理员点击禁用
  → UPDATE iam.accounts SET status = disabled
  → DEL Redis: iam:token:rt:{uid}:*
  → SET Redis: iam:token:revoked:{jti}
  → DEL Redis: iam:perms:{uid}:*
  → 发 Kafka: {env}_iam_user_suspended {uid}
  → 写审计日志（Outbox → MongoDB）

子系统消费 → 清除 L1 → 用户 401 → 无法重新登录
```


---

## 5. 应用架构

### 5.1 项目清单

遵循团队《命名与日志规范手册》四段式命名：\[业务域\]-\[子系统\]-\[功能/模块\]-\[类型\]

| 项目名称 | 类型后缀 | 职责  | 技术栈 |
|------|------|-----|-----|
| iam-center-core-service | -service | 核心业务服务：账号/角色/菜单 CRUD、AT/RT 颁发、Redis 写入、Kafka 生产。对外暴露 gRPC，不直接服务浏览器 | Go  |
| iam-center-admin-service | -service | 管理后台 BFF：面向 iam-center-admin-web 的 HTTP REST API，参数校验、权限检查，调用 core gRPC，自身不写数据库 | Go  |
| iam-center-admin-web | -web | 管理后台前端：账号管理、角色管理、系统管理、审计日志等管理界面 | React |

SDK 说明： 子系统集成能力植入团队现有 common-lib，分为 Go 模块（后端）和 TypeScript 模块（前端）两部分，各接入系统引用对应模块即可，不独立成项目。

### 5.2 项目结构（以 iam-center-core-service 为例）

```
iam-center-core-service/
├── cmd/iam-core/
├── api/iam/v1/
│   ├── iam.proto
│   ├── iam.pb.go
│   └── iam_grpc.pb.go
├── internal/
│   ├── biz/
│   │   ├── account.go
│   │   ├── role.go
│   │   └── menu.go
│   ├── data/
│   │   ├── account.go
│   │   ├── role.go
│   │   └── ent/
│   ├── service/
│   │   └── iam.go
│   └── server/
├── buf.yaml
├── buf.gen.yaml
└── Makefile
```

### 5.3 common-lib IAM 模块拆分

common-lib 中的 IAM 模块按语言拆分为两个子模块，共享同一套 menu_key 语义，前后端权限声明严格对齐：

| 模块  | 语言  | 职责  |
|-----|-----|-----|
| common-lib/iam（Go） | Go  | AuthMiddleware、RequireMenu()、系统注册（bootstrap hook）、Kafka 消费（权限变更/账号禁用）、L1 进程内存缓存（sync.Map） |
| common-lib/iam（TypeScript） | TypeScript | RouteGuard、MenuGuard 组件、canAccess()、权限初始化、401 信号处理与权限刷新 |

menu_key 声明方式（前后端对齐）：

```
后端（config.yaml）                    前端（路由配置）
────────────────────                   ────────────────────────────
menus:                                 {
  - key: risk:users:list                 path: '/users/list',
    name: 用户列表                        menuKey: 'risk:users:list',
    path: /users/list                     component: UserList
                                       }
```

同一个 menu_key 在后端用于接口拦截，在前端用于路由守卫和组件渲染，语义完全一致，由各子系统统一维护，不允许前后端各自定义一套。

前端 SDK 核心能力：

```typescript
// 1. 权限初始化（从 AT payload 或 API 获取 menus[]）
iamClient.init(accessToken)

// 2. 权限查询
iamClient.canAccess('risk:users:list')   // → boolean

// 3. 获取可访问菜单列表（用于导航渲染）
iamClient.getAccessibleMenus()           // → Menu[]

// 4. 路由守卫（React Router 接入）
<RouteGuard menuKey="risk:users:list">
  <UserListPage />
</RouteGuard>
// 无权限 → 自动跳转 403 页面

// 5. 组件级守卫（条件渲染，无权限时不渲染，非置灰）
<MenuGuard menuKey="risk:users:list">
  <DeleteButton />
</MenuGuard>

// 6. 权限变更响应（收到后端 401 时自动触发）
iamClient.onUnauthorized(() => {
  // 重新登录 → 获取新 AT → 重新 init → 界面更新
})
```

前后端权限一致性保证：

```
用户访问 /users/list
  → 前端 RouteGuard：canAccess("risk:users:list") = false → 跳 403，不发请求
  → 或 = true → 放行，发起 API 请求

API 请求到达后端
  → AuthMiddleware：RequireMenu(ctx, "risk:users:list") → 无权 → 403
  → 或有权 → 处理请求

权限变更后
  → 后端 AT 黑名单 → 下次请求 401
  → 前端 onUnauthorized 触发 → 重新登录 → 新 AT → iamClient.init() → 界面重新渲染
  → 前后端权限状态同步，无割裂体验
```

### 5.4 职责边界

```
iam-center-core-service       ← 所有数据写入，权限计算，AT/RT 颁发，Kafka 生产，Redis 独占写入
                                对外暴露 gRPC，不直接处理 HTTP 请求
iam-center-admin-service      ← 管理后台 HTTP REST API（服务 iam-center-admin-web）
                                参数校验 + 权限检查，调用 core gRPC，不写数据库
iam-center-admin-web          ← 管理后台 React 前端，对接 admin-service REST API
common-lib/iam（Go）          ← 后端集成：AuthMiddleware、RequireMenu、系统注册、Kafka 消费、L1 缓存
common-lib/iam（TypeScript）  ← 前端集成：RouteGuard、MenuGuard、canAccess、权限初始化、401 处理
```


---

## 6. 系统架构

### 6.1 基础设施

| 组件  | 选型  | 用途  |
|-----|-----|-----|
| 服务网格 | Istio / K8s | 服务发现、mTLS、流量治理 |
| 消息队列 | Apache Kafka | 双向权限同步事件（Topic 由 IDP 统一管控） |
| 共享缓存 | Redis Cluster | 权限包、AT 黑名单、会话（Key 由 iam-center-core-service 独占写入） |
| 关系数据库 | PostgreSQL | 账号、角色、菜单、权限主数据（Schema: iam） |
| 审计存储 | MongoDB | 审计日志（append-only collection：`iam_audit_logs`） |
| 链路追踪 | Grafana Tempo + OTel | 分布式追踪，trace_id/span_id 全链路透传（W3C Trace Context 标准） |
| 日志  | Grafana Loki | 结构化 JSON 日志采集，字段遵循团队日志规范 |
| 监控告警 | Prometheus + Grafana | 指标采集与可视化 |

### 6.2 部署规范

* 所有服务容器化，部署于 Kubernetes，最小副本数 3，跨可用区反亲和调度
* Redis Cluster 跨可用区部署，3 主 3 从
* PostgreSQL Patroni 主从，自动 Failover

### 6.3 权限缓存分层

| 层级  | 位置  | 延迟  | TTL | 写入方 | 失效方式 |
|-----|-----|-----|-----|-----|------|
| L0 主AT验签 | 进程内 JWKS 公钥 | < 1ms | —   | —   | JWKS 每 30 天轮换 |
| L1 进程内存 | 子系统本地（sync.Map，via common-lib Go） | < 1ms | 30s | common-lib/iam（Go） | Kafka 事件触发主动清除 |
| L2 共享 Redis | iam-center-core-service 独占写 | < 1ms | 5min | iam-center-core-service | 权限变更时 DEL |
| L3 PostgreSQL | iam-center-core-service 主库 | < 10ms | —   | iam-center-core-service | —    |

分级说明：

* 正常请求：L0 验签 + L1 权限，零外部调用，延迟 < 1ms
* L1 Miss → 查 L2 Redis，回填 L1
* L2 Miss（冷启动 / TTL 到期）→ gRPC 调 iam-center-core-service GetUserMenus，查 L3，写入 L2，回填 L1
* L2 由 iam-center-core-service 独占写入，子系统 common-lib 无写权限


---

## 7. 功能模块设计

### 7.1 账号管理

| 功能  | 描述  | 权限要求 |
|-----|-----|------|
| 账号列表 | 搜索（姓名/邮箱/角色/状态）、分页 | 登录可查 |
| 创建账号 | 邮箱（@company.com 域）+ 选角色，发激活邮件 | super_admin |
| 编辑角色分配 | 增减角色，立即触发权限下发 | 向下授权原则 |
| 启用 / 禁用 | 禁用立即踢出所有系统 | super_admin |
| 账号详情 | 基础信息、角色、可访问系统与菜单树、登录记录 | 本人或 super_admin |

账号创建流程：


1. 管理员填写邮箱（Google Workspace 域内）+ 选择角色
2. 系统生成 ID（BIGINT GENERATED ALWAYS AS IDENTITY），写入 iam.accounts，状态 pending
3. 发送激活邮件
4. 员工 Google SSO 登录，状态 pending → active
5. 引导绑定 MFA（YubiKey，后续规划，暂不强制）

### 7.2 角色管理

| 功能  | 描述  | 权限要求 |
|-----|-----|------|
| 角色列表 | 名称、描述、涉及系统数、菜单权限数、成员数 | 登录可查 |
| 创建角色 | 名称、描述，从菜单树勾选权限（按系统分组） | super_admin |
| 编辑角色 | 修改菜单权限，立即对所有成员触发下行同步 | super_admin |
| 删除角色 | 有成员时阻止，需先移除 | super_admin |
| 角色详情 | 菜单权限树（跨系统展示）、成员列表 | 登录可查 |

内置角色（is_builtin = true，不可删除）：

| 角色名 | 说明  |
|-----|-----|
| super_admin | 所有系统全部菜单，管理用户中台本身 |
| readonly_audit | 仅查看账号列表和审计日志 |

### 7.3 系统管理

| 功能  | 描述  |
|-----|-----|
| 系统列表 | 系统名、system_id、状态、菜单数、最后同步时间、SDK 版本 |
| 系统详情 | 完整菜单树（含 deprecated 标记）、引用该菜单的角色列表、接入指引链接 |
| 手动触发同步 | 运维场景下强制重新拉取子系统菜单，不依赖服务重启 |
| 启用 / 禁用 | 禁用后该系统所有登录返回 403 |
| 废弃菜单告警 | 列出所有 deprecated 菜单及引用角色，引导手动清理 |

#### 7.3.1 密钥管理（JWKS 公私钥）

生产环境 AT 签名采用 ES256（ECDSA P-256）非对称算法：私钥由 iam-center-core-service 持有并签发 AT，公钥通过 JWKS 端点对外发布，子系统本地缓存公钥做验签，私钥永不对外暴露。

| 功能  | 描述  |
|-----|-----|
| 密钥列表 | 查看当前所有 kid、算法、状态（active / grace / revoked）、创建时间、过期时间 |
| 手动轮换 | 生成新 EC P-256 密钥对，旧密钥进入 7 天宽限期继续验签（覆盖存活 AT 生命周期） |
| 撤销密钥 | 宽限期结束后手动撤销，撤销后使用该 kid 签发的 AT 全部失效 |
| JWKS 端点 | `GET /.well-known/jwks.json`，无需鉴权，子系统启动时自动拉取，每 30 天轮换 |

#### 7.3b JWKS 公私钥管理

IAM Center 使用 RS256 非对称算法签发 AT，私钥由 IAM Center 安全持有，公钥通过 JWKS 端点对外暴露，网关和各微服务通过拉取公钥完成本地验签。

| 功能  | 描述  |
|-----|-----|
| 公钥列表 | 展示当前所有公钥（kid、算法、状态：current / retiring / expired、生效时间、过期时间） |
| 生成新密钥对 | 管理员触发生成新密钥对；新私钥立即用于签发 AT，老私钥进入 retiring 状态继续验证存量 AT |
| 密钥轮换策略 | 配置 retiring 状态保留时长（默认 2h，覆盖最长 AT 有效期 15min 的余量） |
| 手动吊销密钥 | 将指定 kid 立即置为 expired，持有该 kid 的存量 AT 全部失效（高危，需二次确认） |
| 轮换历史 | 所有密钥操作的审计记录（生成时间、吊销时间、操作人） |

设计原则：私钥永不离开 IAM Center，后台仅展示公钥信息（n/e 参数），不展示任何私钥内容。

### 7.4 个人设置

| 功能  | 说明  |
|-----|-----|
| 个人信息 | 姓名、邮箱、角色、可访问系统列表 |
| MFA 管理 | YubiKey 绑定 / 重置（后续规划，暂不强制） |
| 设备管理 | 已登录设备列表，手动踢出指定设备 |
| 登录记录 | 最近 30 天（时间、system_id、IP、设备） |

### 7.5 审计日志

存储： MongoDB append-only collection，保留 2 年。

| 事件类型 | 核心记录字段 |
|------|--------|
| 账号创建 / 启用 / 禁用 | operator_uid, target_uid, action, created_at |
| 账号角色变更 | operator_uid, target_uid, added_roles\[\], removed_roles\[\] |
| 角色创建 / 编辑 / 删除 | operator_uid, role_id, before_menus\[\], after_menus\[\] |
| 菜单同步 | system_id, added_keys\[\], removed_keys\[\], deprecated_keys\[\] |
| 登录成功 / 失败 | uid, system_id, ip, device_fingerprint, result |
| AT 黑名单写入 | uid, jti, reason, triggered_by |


---

## 8. 接口清单

### 8.1 登录流程（浏览器重定向）

完整登录时序：子系统检测未登录 → 跳转 `/auth/google/init` → Google OAuth → 颁发 AT+RT → 回跳子系统（MFA 暂不强制，YubiKey 后续规划）。

#### 8.1.1 发起 Google SSO

```
GET /auth/google/init
```

**Query 参数：**

| 参数  | 类型  | 必填  | 说明  |
|-----|-----|-----|-----|
| system_id | string | ✅   | 发起登录的子系统标识，颁发 AT 时写入 payload，权限包按此系统加载 |
| redirect_uri | string | ✅   | 登录成功后回跳地址，必须在 iam-center 白名单内，由 common-lib/iam(TS) 自动传入 |
| state | string | ✅   | CSRF 防护随机值，由 common-lib/iam(TS) 生成，回调时原样返回校验 |

**响应：**

```
HTTP 302 Found
Location: https://accounts.google.com/o/oauth2/v2/auth
  ?client_id=<GOOGLE_CLIENT_ID>
  &redirect_uri=https://iam.internal/auth/google/callback
  &response_type=code
  &scope=openid%20email%20profile
  &state=<state>
  &nonce=<nonce>
```

**错误响应：**

| HTTP 状态码 | 错误码 | 说明  |
|----------|-----|-----|
| 400      | INVALID_SYSTEM_ID | system_id 未注册或已禁用 |
| 400      | INVALID_REDIRECT_URI | redirect_uri 不在白名单 |

#### 8.1.2 Google 回调 + 颁发 AT/RT

```
GET /auth/google/callback
```

**Google 回传 Query 参数（由 Google 附加，非客户端传入）：**

| 参数  | 类型  | 说明  |
|-----|-----|-----|
| code | string | Google 授权码，iam-center 用此换取 id_token |
| state | string | 与 8.1.1 的 state 一致，服务端校验 CSRF |

**内部处理流程：**

```
1. 用 code 换取 Google id_token，验签并解析 sub / email
2. 查 iam.accounts（by google_sub）
   ├─ 不存在 / disabled → 403
   └─ pending → 激活账号
3. 检查是否已有有效 session（SSO 复用直接续签）
4. 颁发 AT + RT（MFA 暂不强制，YubiKey 后续规划）
```

MFA 验证接口（暂未启用，YubiKey 后续规划）：

```json
POST /auth/totp/verify
Content-Type: application/json

{
  "yubikey_otp": "123456",
  "session_token": "<临时 session token，由步骤 5 颁发>"
}
```

**颁发成功响应：**

```
HTTP 200 OK
Set-Cookie: iam_rt=<refresh_token>; HttpOnly; Secure; SameSite=Strict; Path=/auth; Max-Age=2592000
Content-Type: application/json

{
  "access_token": "<JWT>",
  "token_type": "Bearer",
  "expires_in": 900,
  "system_id": "trade"
}
```

随即 302 回跳 redirect_uri，AT 通过 URL fragment 或 PostMessage 传回 common-lib/iam(TS)。

**AT Payload（JWT Claims）：**

```json
{
  "iss": "https://iam.internal",
  "sub": "12345",
  "email": "alice@company.com",
  "jti": "550e8400-e29b-41d4-a716-446655440000",
  "system_id": "trade",
  "iat": 1704067200,
  "exp": 1704070800
}
```

AT 有效期 15 分钟，仅含身份信息，不含菜单权限；权限包独立存于 Redis `iam:perms:{uid}:{system_id}`，子系统通过 common-lib 加载。

**错误响应：**

| HTTP 状态码 | 错误码 | 说明  |
|----------|-----|-----|
| 403      | ACCOUNT_PENDING | 账号存在但未被管理员激活 |
| 403      | ACCOUNT_DISABLED | 账号已禁用 |
| 403      | SYSTEM_DISABLED | 目标系统已禁用 |
| 401      | MFA_INVALID | MFA 验证失败（后续规划，YubiKey） |
| 401      | MFA_MAX_RETRY | MFA 连续失败超过 5 次，账号临时锁定 10 分钟（后续规划） |
| 400      | STATE_MISMATCH | state 校验失败，疑似 CSRF |

#### 8.1.3 刷新 AT

```
POST /auth/refresh
Content-Type: application/json
Authorization: Bearer <当前 AT（可已过期）>
```

**Request Body：**

```json
{
  "refresh_token": "<RT 字符串，或由 HttpOnly Cookie iam_rt 自动携带>",
  "system_id": "trade"
}
```

| 字段  | 类型  | 必填  | 说明  |
|-----|-----|-----|-----|
| refresh_token | string | ✅（Cookie 方式可省略 body） | RT，有效期 30 天，单设备单 Token |
| system_id | string | ✅   | 指定为哪个子系统颁发新 AT |

**成功响应：**

```json
{
  "access_token": "<新 JWT>",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**错误响应：**

| HTTP 状态码 | 错误码 | 说明  |
|----------|-----|-----|
| 401      | RT_EXPIRED | RT 已过期（30 天），需重新登录 |
| 401      | RT_REVOKED | RT 已被吊销（账号禁用或主动登出） |
| 401      | ACCOUNT_DISABLED | 刷新时账号已被禁用 |
| 429      | RATE_LIMITED | 超过限流：10 次/min（账号级） |

#### 8.1.4 登出

```
POST /auth/logout
Authorization: Bearer <AT>
Content-Type: application/json
```

**Request Body：**

```json
{
  "device_id": "device-fingerprint-abc123",
  "all_devices": false
}
```

| 字段  | 类型  | 必填  | 说明  |
|-----|-----|-----|-----|
| device_id | string | ❌   | 指定登出的设备，为空则登出当前设备 |
| all_devices | boolean | ❌   | true = 踢出该账号所有设备（默认 false） |

**成功响应：**

```
HTTP 200 OK
Set-Cookie: iam_rt=; HttpOnly; Secure; Max-Age=0

{ "ok": true }
```

内部动作：DEL `iam:token:rt:{uid}:{device_id}` → SET `iam:token:revoked:{jti}` → 写审计日志。

#### 8.1.5 JWKS 公钥下发

```
GET /.well-known/jwks.json
```

无需鉴权。子系统启动时由 common-lib/iam(Go) 拉取并缓存，每 30 天轮换。

**响应：**

```json
{
  "keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "kid": "2025-01-key-1",
      "use": "sig",
      "alg": "ES256",
      "x": "<base64url>",
      "y": "<base64url>"
    }
  ]
}
```

签名算法采用 ES256（ECDSA P-256），比 RS256 密钥更短、验签更快，AT 本地验签延迟 < 1ms。

### 8.2 内部 gRPC（子系统调用，L2 Miss 兜底）

| 服务  | 方法  | 描述  | 调用时机 |
|-----|-----|-----|------|
| iam-center-core-service | GetUserMenus(uid, system_id) | 查询账号菜单权限 | L2 Redis Miss 时 |
| iam-center-core-service | ValidateToken(token) | 验证 AT 有效性（含黑名单） | 主动验证场景 |
| iam-center-core-service | GetUserInfo(uid) | 账号基础信息 | 业务查询 |

### 8.3 管理后台 REST API（需管理员主AT）

所有接口均需在请求头携带管理员 AT：`Authorization: Bearer <AT>`

**账号管理：**

| 方法  | 路径  | 描述  |
|-----|-----|-----|
| GET | /api/v1/admin/users | 账号列表，支持 Query 参数：`keyword`（姓名/邮箱）、`role_id`、`status`、`page`、`page_size` |
| POST | /api/v1/admin/users | 创建账号，Body：`{ email, name, role_ids[] }` |
| GET | /api/v1/admin/users/{id} | 账号详情，含角色列表和可访问系统 |
| PATCH | /api/v1/admin/users/{id}/status | 启用/禁用账号，Body：`{ status: 2\|3 }` |
| PATCH | /api/v1/admin/users/{id}/roles | 更新账号角色，Body：`{ role_ids[] }` |

**角色管理：**

| 方法  | 路径  | 描述  |
|-----|-----|-----|
| GET | /api/v1/admin/roles | 角色列表，支持 Query：`keyword`、`page`、`page_size` |
| POST | /api/v1/admin/roles | 创建角色，Body：`{ name, description, menu_keys[] }` |
| GET | /api/v1/admin/roles/{id} | 角色详情，含完整菜单权限列表 |
| PATCH | /api/v1/admin/roles/{id} | 更新角色名称/描述/菜单，Body：`{ name?, description?, menu_keys[]? }` |
| DELETE | /api/v1/admin/roles/{id} | 删除角色（内置角色不可删，返回 403） |

**系统管理：**

| 方法  | 路径  | 描述  |
|-----|-----|-----|
| GET | /api/v1/admin/systems | 系统列表，含菜单数、最后同步时间 |
| GET | /api/v1/admin/systems/{id} | 系统详情，含完整菜单树（含 deprecated 标记）、SDK 版本、接入状态 |
| POST | /api/v1/admin/systems/{id}/sync | 手动触发菜单重新同步（运维场景） |
| PATCH | /api/v1/admin/systems/{id}/status | 启用/禁用系统，Body：`{ status: 1\|2 }` |

**系统接入申请：**

| 方法  | 路径  | 描述  |
|-----|-----|-----|
| POST | /api/v1/admin/system-requests | 提交接入申请，Body：`{ system_id, name, owner, purpose }` |
| GET | /api/v1/admin/system-requests | 申请列表，Query：status（pending/approved/rejected） |
| PATCH | /api/v1/admin/system-requests/{id}/approve | 审批通过，返回 hmac_secret（dev/test）或 JWKS URL（prod） |
| PATCH | /api/v1/admin/system-requests/{id}/reject | 驳回申请，Body：`{ reason }` |

**密钥管理（JWKS 公私钥）：**

| 方法  | 路径  | 描述  |
|-----|-----|-----|
| GET | /api/v1/admin/keys | 当前所有签名密钥列表（kid、算法、状态、创建时间、过期时间） |
| POST | /api/v1/admin/keys/rotate | 手动触发密钥轮换，生成新 EC P-256 密钥对，旧密钥进入宽限期（7天）继续验签 |
| DELETE | /api/v1/admin/keys/{kid} | 撤销指定密钥（宽限期结束后执行，确保无存活 AT 使用该 kid） |
| GET | /.well-known/jwks.json | 公开 JWKS 端点（无需鉴权），子系统拉取公钥用于本地验签 |

**审计日志：**

| 方法  | 路径  | 描述  |
|-----|-----|-----|
| GET | /api/v1/admin/audit-logs | 审计日志列表，支持 Query：`operator_uid`、`target_uid`、`action`、`start_at`、`end_at`、`page`、`page_size` |


---

## 9. 数据库设计

遵循团队《数据库设计与变更规范》（V1.0.0）。

### 9.1 Schema 规划

```sql
CREATE SCHEMA IF NOT EXISTS iam;
```

### 9.2 DDL 说明

* 主键：BIGINT GENERATED ALWAYS AS IDENTITY
* 禁止数据库级 FOREIGN KEY，关联关系由应用层维护
* 仅软删除：deleted_at TIMESTAMPTZ(6)，禁止物理删除
* 所有表必须包含：created_at、updated_at、deleted_at、created_by、updated_by
* 时间字段统一 TIMESTAMPTZ(6)，存 UTC
* 禁止表名前缀（无 t_），禁止使用 SQL 保留字（users → accounts）

### 9.3 实体关系图（ER 图）

```
erDiagram
    accounts {
        bigint      id            PK  "主键"
        varchar255  email             "企业邮箱，唯一"
        varchar128  name              "员工姓名"
        varchar255  google_sub        "Google OIDC sub，唯一不变"
        smallint    user_type         "1=INTERNAL"
        varchar64   platform_id       "1期固定 system"
        smallint    status            "1=pending 2=active 3=disabled"
        timestamptz last_login_at     "最近登录时间"
    }
    systems {
        bigint      id            PK  "主键"
        varchar64   system_id         "子系统唯一标识"
        varchar128  display_name      "显示名称"
        varchar64   checksum          "菜单列表 MD5"
        smallint    status            "1=active 2=disabled"
        timestamptz last_sync_at      "最近菜单同步时间"
    }
    system_menus {
        bigint      id            PK  "主键"
        varchar64   system_id     FK  "所属子系统"
        varchar128  menu_key          "权限键 {system_id}:{模块}:{页面}"
        varchar128  name              "菜单名称"
        varchar255  path              "前端路由路径"
        varchar128  parent_key        "父菜单 key，NULL=顶级"
        int         sort_order        "排序权重"
        smallint    status            "1=active 2=deprecated 3=deleted"
    }
    roles {
        bigint      id            PK  "主键"
        varchar128  name              "角色名称，唯一"
        text        description       "角色职责描述"
        boolean     is_builtin        "内置角色不可删改"
    }
    role_menus {
        bigint      id            PK  "主键"
        bigint      role_id       FK  "关联 roles.id"
        varchar64   system_id         "菜单所属子系统（冗余）"
        varchar128  menu_key          "关联 system_menus.menu_key"
    }
    account_roles {
        bigint      id            PK  "主键"
        bigint      account_id    FK  "关联 accounts.id"
        bigint      role_id       FK  "关联 roles.id"
        bigint      granted_by        "授权人 accounts.id"
    }
    account_devices {
        bigint      id            PK  "主键"
        bigint      account_id    FK  "关联 accounts.id"
        varchar64   device_id         "设备指纹，前端 SDK 生成"
        varchar128  device_name       "设备名称"
        inet        last_ip           "最近登录 IP"
        timestamptz last_seen_at      "最近活跃时间"
    }
    user_menu_cache {
        bigint      id            PK  "主键"
        bigint      uid           FK  "关联 accounts.id"
        varchar64   system_id     FK  "关联 systems.system_id"
        varchar128  menu_key          "被授权的菜单 key"
        timestamptz synced_at         "最后同步时间 >15min 视为过期"
    }
    accounts      ||--o{ account_roles   : "被分配角色"
    roles         ||--o{ account_roles   : "被分配给账号"
    accounts      ||--o{ account_devices : "持有设备"
    roles         ||--o{ role_menus      : "包含菜单权限"
    systems       ||--o{ system_menus    : "上报菜单"
    system_menus  ||--o{ role_menus      : "被角色引用"
    accounts      ||--o{ user_menu_cache : "权限本地缓存"
    systems       ||--o{ user_menu_cache : "缓存归属系统"
    system_menus  }o--o| system_menus    : "父子菜单 parent_key"
```

说明：

* `account_roles.granted_by` 指向 `accounts.id`（授权操作人），与 `account_id` 同表不同语义
* `role_menus.menu_key` 为逻辑外键，未设物理约束，允许引用 `deprecated` 状态菜单（配套告警机制）
* `user_menu_cache` 为 L4 兜底缓存，仅在 Redis L2 与 gRPC L3 均不可用时生效，由 common-lib/iam(Go) 自动维护

### 9.4 核心表 DDL

```sql
-- ============================================================
-- 账号表：对应一个真实内部员工，与 Google Workspace 账号一一绑定
-- ============================================================
CREATE TABLE iam.accounts (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         VARCHAR(255)   NOT NULL,
    name          VARCHAR(128)   NOT NULL DEFAULT '',
    google_sub    VARCHAR(255)   NOT NULL,
    user_type     SMALLINT       NOT NULL DEFAULT 1,
    platform_id   VARCHAR(64)    NOT NULL DEFAULT 'system',
    status        SMALLINT       NOT NULL DEFAULT 1,
    last_login_at TIMESTAMPTZ(6),
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_accounts_email      UNIQUE (email),
    CONSTRAINT uq_accounts_google_sub UNIQUE (google_sub)
);
COMMENT ON TABLE  iam.accounts            IS '内部员工账号表，与 Google Workspace 一一绑定，1期唯一登录方式为 Google SSO + MFA（后续规划）';
COMMENT ON COLUMN iam.accounts.status     IS '1=pending 待激活  2=active 正常使用  3=disabled 已禁用（触发吊销链：RT 全清 → AT 黑名单 → Kafka user.suspended）';
COMMENT ON COLUMN iam.accounts.user_type  IS '1=INTERNAL 内部员工（1期）；2期扩展 PLATFORM 平台用户';
COMMENT ON COLUMN iam.accounts.platform_id IS '平台归属：1期固定 system；2期多联盟站时区分各平台';
COMMENT ON COLUMN iam.accounts.google_sub IS 'Google OIDC sub 字段，账号迁移/邮箱改名后仍唯一标识同一人';

-- ============================================================
-- 子系统注册表：每个内部系统启动时自动 upsert，由 common-lib/iam(Go) 完成
-- ============================================================
CREATE TABLE iam.systems (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    system_id     VARCHAR(64)    NOT NULL,
    display_name  VARCHAR(128)   NOT NULL,
    checksum      VARCHAR(64),
    status        SMALLINT       NOT NULL DEFAULT 1,
    last_sync_at  TIMESTAMPTZ(6),
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_systems_system_id UNIQUE (system_id)
);
COMMENT ON TABLE  iam.systems             IS '子系统注册表，系统启动时由 common-lib/iam(Go) 自动 upsert，checksum 变化时触发菜单 diff';
COMMENT ON COLUMN iam.systems.system_id   IS '子系统标识符，全局唯一，声明于项目 config.yaml，menu_key 前缀必须与此一致';
COMMENT ON COLUMN iam.systems.checksum    IS '菜单列表 MD5，上行同步时对比此值，一致则跳过 diff，减少无效写入';
COMMENT ON COLUMN iam.systems.status      IS '1=active  2=disabled（禁用后该系统所有用户登录均返回 403）';

-- ============================================================
-- 菜单表：权限最小粒度（1期），由子系统定义并上报，用户中台存储管理
-- ============================================================
CREATE TABLE iam.system_menus (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    system_id     VARCHAR(64)    NOT NULL,
    menu_key      VARCHAR(128)   NOT NULL,
    name          VARCHAR(128)   NOT NULL,
    path          VARCHAR(255)   NOT NULL DEFAULT '',
    parent_key    VARCHAR(128),
    sort_order    INT            NOT NULL DEFAULT 0,
    status        SMALLINT       NOT NULL DEFAULT 1,
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_system_menus_key UNIQUE (system_id, menu_key)
);
COMMENT ON TABLE  iam.system_menus            IS '菜单定义表，1期权限最小粒度，由子系统启动时上报，用户中台做 diff 维护；menu_key 与前端路由强绑定';
COMMENT ON COLUMN iam.system_menus.menu_key   IS '格式：{system_id}:{模块}:{页面}，例如 trade:order:list；前后端必须一致，CI 静态检查拦截不匹配';
COMMENT ON COLUMN iam.system_menus.parent_key IS '父菜单 menu_key，NULL=顶级菜单；父子权限独立，不自动继承';
COMMENT ON COLUMN iam.system_menus.status     IS '1=active 正常  2=deprecated 已废弃但有角色引用（告警，等待管理员手动清理）  3=deleted 已安全删除';

-- ============================================================
-- 角色表：菜单权限的命名集合，可跨系统组合菜单
-- ============================================================
CREATE TABLE iam.roles (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name          VARCHAR(128)   NOT NULL,
    description   TEXT,
    is_builtin    BOOLEAN        NOT NULL DEFAULT FALSE,
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_roles_name UNIQUE (name)
);
COMMENT ON TABLE  iam.roles            IS '角色定义表，角色是菜单权限的命名集合，可跨系统组合；遵循向下授权原则，管理员只能分配自己已有的权限';
COMMENT ON COLUMN iam.roles.is_builtin IS 'TRUE=系统内置角色（如 super_admin），不允许通过管理后台删除或编辑菜单';

-- ============================================================
-- 角色-菜单关联表：记录角色包含哪些菜单权限项
-- ============================================================
CREATE TABLE iam.role_menus (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    role_id       BIGINT         NOT NULL,
    system_id     VARCHAR(64)    NOT NULL,
    menu_key      VARCHAR(128)   NOT NULL,
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_role_menus UNIQUE (role_id, system_id, menu_key)
);
COMMENT ON TABLE  iam.role_menus           IS '角色与菜单的 M:N 关联表；menu_key 对应 system_menus 中 status=deprecated 的记录时触发告警，需管理员介入';
COMMENT ON COLUMN iam.role_menus.system_id IS '冗余字段，与 menu_key 共同标识菜单，用于按系统批量查询角色权限，无需 JOIN system_menus';
COMMENT ON COLUMN iam.role_menus.menu_key  IS '逻辑外键，不设物理外键约束，允许 deprecated 状态的菜单继续被引用（告警处理）';

-- ============================================================
-- 账号-角色关联表：记录员工被分配了哪些角色
-- ============================================================
CREATE TABLE iam.account_roles (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    account_id    BIGINT         NOT NULL,
    role_id       BIGINT         NOT NULL,
    granted_by    BIGINT         NOT NULL,
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_account_roles UNIQUE (account_id, role_id)
);
COMMENT ON TABLE  iam.account_roles           IS '账号与角色的 M:N 关联表，变更后触发权限下行同步：写 PG → DEL Redis → AT 黑名单 → Kafka 通知子系统';
COMMENT ON COLUMN iam.account_roles.granted_by IS '执行授权的管理员 accounts.id，用于审计追溯；向下授权原则：管理员只能将自己已持有的角色授予他人';

-- ============================================================
-- 账号设备表：记录员工已登录的设备，支持手动踢出指定设备
-- ============================================================
CREATE TABLE iam.account_devices (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    account_id    BIGINT         NOT NULL,
    device_id     VARCHAR(64)    NOT NULL,
    device_name   VARCHAR(128),
    last_ip       INET,
    last_seen_at  TIMESTAMPTZ(6),
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_account_devices UNIQUE (account_id, device_id)
);
COMMENT ON TABLE  iam.account_devices             IS '账号设备登录记录，个人设置页"设备管理"的数据源；踢出设备时 DEL 对应 Redis RT Key，AT 加黑名单';
COMMENT ON COLUMN iam.account_devices.device_id   IS '设备指纹，由前端 common-lib/iam(TS) 生成，格式由 SDK 规范，不可伪造';
COMMENT ON COLUMN iam.account_devices.last_ip     IS '使用 PostgreSQL 原生 INET 类型，支持 IPv4/IPv6，便于后续 IP 风控查询';

-- ============================================================
-- 用户菜单缓存表（L4 兜底，由 common-lib/iam(Go) 自动建表并维护）
-- 仅在 Redis L2 和 gRPC L3 均不可用时作为最终降级手段
-- ============================================================
CREATE TABLE iam.user_menu_cache (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    uid           BIGINT         NOT NULL,
    system_id     VARCHAR(64)    NOT NULL,
    menu_key      VARCHAR(128)   NOT NULL,
    synced_at     TIMESTAMPTZ(6) NOT NULL,
    created_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ(6),
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64),
    CONSTRAINT uq_user_menu_cache UNIQUE (uid, system_id, menu_key)
);
COMMENT ON TABLE  iam.user_menu_cache           IS '子系统本地菜单权限缓存（L4 兜底），由 common-lib/iam(Go) 自动建表并维护；仅在 Redis L2 与 gRPC L3 均不可用时使用';
COMMENT ON COLUMN iam.user_menu_cache.synced_at IS '最后同步时间，超过 15 分钟视为过期，触发异步重新加载';
```

### 9.5 Migration 管理

* 工具：golang-migrate 或 goose
* 文件位置：migrations/，命名格式 {序号}_{描述}.up.sql / .down.sql
* 新增表：Tech Lead 审批；修改字段类型：Tech Lead + DBA 审批
* 大表变更（> 100 万行）：DBA 专项评审，使用 gh-ost 执行


---

## 10. 边界与异常处理

### 10.1 认证异常

| 场景  | 处理策略 |
|-----|------|
| Google SSO 不可用 | 登录失败，展示错误页，不提供降级方式 |
| AT 签名验证失败 | 返回 401，客户端清除 token 重新登录 |
| AT 在黑名单中 | 返回 401，前端 onUnauthorized 触发重新登录 |
| RT 已过期 | 返回 401，跳登录页 |
| RT 重放攻击 | 吊销该 uid 所有 RT，所有设备强制下线，写审计日志 |
| 账号 disabled | 返回 403，展示"账号已被停用，请联系管理员" |
| 账号无该系统访问权限 | 返回 403，展示"您无权访问该系统" |

### 10.2 权限同步异常

| 场景  | 处理策略 |
|-----|------|
| 子系统启动时 Kafka 不可用 | 重试 3 次（指数退避 1s/2s/4s），失败后系统正常启动，菜单同步延迟，已有权限不受影响 |
| iam-center-core-service 消费失败 | 消息进死信队列，告警，人工介入重新消费 |
| Redis DEL 失败 | 重试 3 次，失败则等待 TTL 自然过期（最多 5 分钟） |
| Kafka 下行消费失败 | common-lib 重试 3 次，失败记录死信，L1 TTL 到期后从 L2 重新加载 |
| L2 Redis 不可用 | 降级到 L3 gRPC 查询，性能下降但功能不中断 |
| Redis 完全不可用（L2 + 黑名单均无法访问） | 由 `FailOpen` 配置决定降级策略：`true`（默认）= 放行，高可用优先；`false` = 拒绝，安全优先。生产环境建议 `true`，配合 Redis 高可用部署 |
| 菜单 key 冲突 | iam-center-core-service 校验 key 必须以 {system_id}: 为前缀，冲突时拒绝注册并告警 |
| 删除的菜单有角色引用 | 标记 deprecated，不强删，告警，管理员手动清理后置 deleted |
| 前后端 menu_key 不一致 | 前端路由配置的 menuKey 与 config.yaml 中的 menu_key 不匹配，CI 静态检查拦截（common-lib 提供校验工具） |

### 10.3 系统级边界

**白名单分层说明：**

白名单共有两层，各司其职，需分别配置：

| 层级  | 配置位置 | 作用  | 适用场景 |
|-----|------|-----|------|
| 网关层 | `gateway.yaml` 的 `whitelist` | 请求在网关直接放行，不做任何 JWT 验签，不转发鉴权判断 | 健康检查（`/health`、`/readyz`）、Metrics、公开登录接口 |
| SDK 层 | 各服务 `Config.Whitelist` | 请求到达业务服务后跳过 SDK 内部的 JWT 验签，直接进入 Handler | 服务内部公开路径，或本地开发调试时临时跳过鉴权 |

注意：两层白名单独立，都需要配置。只配网关层而不配 SDK 层，服务内部仍会鉴权；只配 SDK 层而不配网关层，网关会在鉴权失败后拦截请求。

**限流策略：**

| 接口  | 账号级 | IP 级 |
|-----|-----|------|
| 登录发起 | 5/min | 20/min |
| Token 刷新 | 10/min | 50/min |
| 管理后台读 | 30/s | 100/s |
| 管理后台写 | 5/s | 20/s |

**熔断策略：**

* iam-center-core-service gRPC：失败率 > 50% 触发熔断，降级返回空权限包，30 秒后半开探测
* Google SSO 回调：不熔断，超时直接失败
* Kafka 生产失败：写入本地 WAL，异步重发

**数据一致性：**

* 权限写入：先写 PostgreSQL，再 DEL Redis
* AT 黑名单：写 Redis 失败重试 3 次；仍失败则写 iam.at_blacklist 持久化，AuthMiddleware 双查
* 审计日志：Outbox 模式，操作与审计同一事务写 outbox 表，relay worker 异步写 MongoDB
* 软删除：所有查询加 WHERE deleted_at IS NULL