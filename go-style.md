---
title: "Go 编码风格规范"
version: "2.0.0-draft"
last_modified: "2026-04-26"
source: "规范版本库0.0.2 / go.md"
---

# Go 编码风格规范

# Go 编码风格规范

> 版本：V2.0.0-draft | 状态：草案 | 适用范围：所有 Go 服务（技术栈：Go 1.24 + Gin + MySQL 8.4 + Redis + Kafka）


---

## 1. 总则

本规范基于 [Effective Go](https://go.dev/doc/effective_go) 与 [Uber Go Style Guide](https://github.com/uber-go/guide)，结合团队服务架构实践制定。所有 Go 项目必须在 CI 中通过 `golangci-lint` 与 `go test` 方可合并。公共能力优先使用公共库 `gin-go-common-lib`（日志、中间件、统一响应、错误、MySQL / Redis / Kafka 客户端、命名、幂等），**不要重复造轮子**。


---

## 2. 项目结构

每个服务统一采用以下目录结构（Gin，分层：handler → biz → data）：

```
trade-service/
├── .github/workflows/ci.yml   — PR / push：lint + test + build
├── Makefile                    — 本地开发快捷命令（build / run / test / lint / migrate）
├── Dockerfile                  — 多阶段构建，distroless 运行时镜像
├── go.mod / go.sum
├── cmd/
│   └── trade/main.go           — 启动入口：读配置 → 构造依赖 → 注册路由 → 优雅退出
├── configs/
│   └── dev/server.yaml         — 本地开发配置（密钥走环境变量，不入库）
├── migrations/                 — goose 迁移（见 database.md §6）
│   ├── 000001_trade_create_orders.up.sql
│   └── 000001_trade_create_orders.down.sql
├── docs/
│   ├── api/                    — 接口文档（api-doc.md 模板）
│   └── error-codes.yaml        — 错误码登记（error-codes.md）
└── internal/
    ├── router/                 — 路由注册、中间件装配（gin.Engine）
    ├── handler/                — HTTP 适配：参数绑定、调用 biz、输出响应；不含业务逻辑
    ├── biz/                    — 领域模型 + Usecase；纯业务，不依赖 gin / gorm
    ├── data/                   — 仓储实现：MySQL（GORM）/ Redis / Kafka；实现 biz 定义的接口
    ├── errcode/                — 本服务业务错误码（error-codes.md §5）
    ├── contract/               — 对外事件载荷结构体（Kafka，见 kafka.md §2.3）
    ├── job/                    — 定时任务、Outbox relay、Kafka 消费者
    └── conf/                   — 配置结构体与加载
```

**强制约束：**

* `internal/` 下严格分层，**依赖方向只能是** `handler → biz → data`，禁止反向依赖；`biz` 通过接口使用 `data`，接口定义在 `biz`（使用方）
* `biz` 包**不得 import** `gin`、`gorm`、`go-redis`、`kafka-go`
* `cmd/` 中仅做依赖装配与启动，不含业务逻辑
* handler 只做三件事：绑定校验 → 调用 usecase → `response.*` 输出
* 跨服务共享代码放入独立 Go Module（如 `gin-go-common-lib`），通过版本化引用
* 服务间**不得直接读写对方的 MySQL 库**，只能走 REST 或 Kafka 事件

### 2.1 公共库使用速查

| 需求 | 用公共库的 |
|---|---|
| Gin 中间件 | `middleware.Default(log, reg)`（TraceID → Recovery → Logger → Metrics） |
| 统一响应 / 绑定校验 | `response.JSON` / `JSONList` / `FailErr` / `BindJSONOrFail` / `BindQueryOrFail` |
| 业务错误 | `errors.AppError`（见 [error-codes.md](error-codes.md)） |
| 日志 | `logging.New(service)`，`log.Info(ctx, msg, args...)` |
| MySQL | `storage/db.NewMySQL(cfg, log)` |
| Redis | `storage/redis.New(cfg)` |
| Kafka | `mq.NewProducer` / `mq.NewConsumer` + `naming.*` |
| 幂等 | `idempotency.NewRedisStore` + `idempotency.Middleware` |
| 金额 | `util/decimalx`（底层 `shopspring/decimal`） |
| 业务 ID（`oid`） | `util/idgen`（UUIDv7，见 [id-generation.md](id-generation.md)）；自增 `id` 由数据库生成，不进 DTO |
| 单进程合并回源 | `cache/singleflight` |

---

## 3. 命名规范

### 3.1 包命名

* 全小写，单个单词，禁止下划线和驼峰：`userorder` ✗ → `order` ✓
* 包名应是名词，避免 `util`、`common`、`helper` 等万能命名
* 包名不应与标准库冲突

### 3.2 变量与函数

* 局部变量用短名：`i`, `ctx`, `err`, `tx`
* 导出函数使用 MixedCaps：`GetUserByID`
* 非导出函数使用 mixedCaps：`parseRequestBody`
* 接收者名使用 1-2 字母缩写：`func (s *Service) CreateOrder()`
* 布尔变量/函数用 `is`、`has`、`can` 前缀：`IsActive()`, `HasPermission()`

### 3.3 接口

* 单方法接口用方法名 + `er` 后缀：`Reader`, `Closer`, `Validator`
* 多方法接口用具体名词：`OrderRepository`, `PaymentGateway`
* 接口放在**使用方**包内，而非实现方

### 3.4 常量与枚举

```go
// 常量组用类型约束
type OrderStatus int

const (
    OrderStatusPending   OrderStatus = iota + 1
    OrderStatusConfirmed
    OrderStatusCancelled
)
```


---

## 4. 错误处理

### 4.1 基本原则

* 永远检查 error，禁止使用 `_` 忽略
* 使用 `fmt.Errorf("描述: %w", err)` 包装错误，保留错误链
* 业务错误使用团队错误码体系：`errors.AppError` + `internal/errcode`（见 [error-codes.md](error-codes.md)）

### 4.2 错误处理模式

```go
// ✓ 正确：立即处理，提前返回
result, err := repo.GetOrder(ctx, id)
if err != nil {
    return nil, fmt.Errorf("get order %s: %w", id, err)
}

// ✗ 错误：嵌套过深
if err == nil {
    // happy path buried in nesting
}
```

### 4.3 Panic 使用

* **禁止**在业务代码中使用 `panic`
* 仅允许在 `init()` 和程序启动阶段对不可恢复错误使用
* HTTP handler 层必须有 recover middleware（`middleware.Recovery`，已含在 `middleware.Default` 中）


---

## 5. 并发编程

* 启动 goroutine 必须确保有退出机制（`context.Context` 或 `done channel`）
* 禁止裸启 goroutine，必须通过 `errgroup` 或封装函数管理生命周期
* channel 优先于 mutex；需要 mutex 时使用最小锁粒度
* 禁止在持锁期间做 I/O 操作

```go
// ✓ 推荐：errgroup 管理并发
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error {
    return fetchPrices(ctx)
})
g.Go(func() error {
    return fetchBalances(ctx)
})
if err := g.Wait(); err != nil {
    return err
}
```


---

## 6. 依赖注入

* 使用**构造函数注入**，禁止包级全局变量持有依赖（数据库、Redis、日志器）
* 构造函数命名 `NewXxx`，参数为接口或具体依赖，返回具体类型（接受接口、返回结构体）
* 依赖在 `cmd/.../main.go` **手动装配**，不引入 Wire 等代码生成工具；依赖多到装配代码难以维护时再评估

```go
type OrderUsecase struct {
	repo   OrderRepository // 接口，定义在 biz 包内
	cache  Cache
	logger *logging.Logger
}

func NewOrderUsecase(repo OrderRepository, cache Cache, logger *logging.Logger) *OrderUsecase {
	return &OrderUsecase{repo: repo, cache: cache, logger: logger}
}
```

---

## 7. 日志规范

* 统一使用公共库 `logging`（基于标准库 `slog`，JSON 输出），禁止 `fmt.Println` / `log.Println` 或字符串拼接
* 日志级别：Debug / Info / Warn / Error，具体见 [naming-logging.md](naming-logging.md)
* 必须传 `ctx`，`trace_id` 由 `middleware.TraceID` 注入、由日志器自动带出
* **禁止**记录 Token、密码、完整证件号 / 卡号、完整请求 / 响应体

```go
log.Info(ctx, "order created",
	"order_oid", order.OID,
	"uid", order.UID,
	"order_amount", order.Amount.String(), // 金额用 decimal 的字符串
)
```

---

## 8. Lint 配置 (.golangci.yml)

项目根目录必须包含统一的 `.golangci.yml`：

```yaml
run:
  timeout: 5m
  go: "1.24"

linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused
    - gofmt
    - goimports
    - misspell
    - revive
    - gosec
    - bodyclose
    - noctx
    - exhaustive

linters-settings:
  revive:
    rules:
      - name: exported
      - name: var-naming
      - name: package-comments
  gosec:
    excludes:
      - G104  # 由 errcheck 覆盖

issues:
  max-issues-per-linter: 0
  max-same-issues: 0
```


---

## 9. 注释规范

### 9.1 基本原则

* 注释使用**中文**，同一项目内保持一致
* 注释应解释 **为什么（Why）**，而非 **做了什么（What）**——代码本身应具备可读性
* 过时的注释比没有注释更有害，修改代码时**必须同步更新注释**
* 禁止提交被注释掉的代码块，使用 Git 历史追溯

### 9.2 包注释

每个包必须有包注释，放在包内任意一个文件的 `package` 声明之上：

```go
// Package order 提供订单域的核心业务逻辑，包括创建、支付、取消等用例。
// 本包为纯业务层，不依赖任何框架和基础设施。
package order
```

* 以 `// Package <包名>` 开头（godoc 约定）
* 简要说明包的职责与边界，1-3 行即可

### 9.3 公开（大写开头）方法注释

所有大写开头的类型、函数、常量、变量属于包的公开 API，**必须**有注释，且以符号名开头（`golint` / `revive` 强制）：

```go
// OrderStatus 表示订单生命周期状态。
type OrderStatus int

// ErrOrderNotFound 在根据 ID 查询订单不存在时返回。
var ErrOrderNotFound = errors.New("order not found")

// MaxRetryCount 是提现签名的最大重试次数。
const MaxRetryCount = 3

// CreateOrder 创建一笔新订单并写入流水。
// 若余额不足返回 ErrInsufficientBalance；若幂等键冲突返回 ErrDuplicateOrder。
func (s *OrderService) CreateOrder(ctx context.Context, req *CreateOrderReq) (*Order, error) {
```

### 9.4 私有（小写开头）方法注释

小写开头的函数/类型仅包内可见，无需强制注释，但以下情况**建议**添加：

* 逻辑复杂度高（圈复杂度 > 10）
* 包含非显而易见的业务规则或算法
* 存在反直觉的实现（性能优化、兼容 hack）

```go
// buildSweepTx 构造归集交易。
// 对 TRON 链需要先检查 Energy 代理余额，不足时走燃烧模式，
// 因此返回值中额外携带 energyFrom 标记。
func (s *sweepBuilder) buildSweepTx(ctx context.Context, addrs []string) (*RawTx, error) {
```

### 9.5 TODO / FIXME / HACK 标记

使用统一格式，便于 CI 统计和追踪：

```go
// TODO(zhangsan): 2025-06 前迁移到新的费率引擎，届时移除硬编码费率
// FIXME(lisi): 并发场景下 balance_after 可能不准确，需加分布式锁
// HACK: TRC20 Transfer 事件有时缺少 from 字段，此处做兼容回填
```

* `TODO` — 待完成的功能或改进，标注负责人和预计时间
* `FIXME` — 已知缺陷，标注负责人
* `HACK` — 临时方案，说明原因和预期移除条件

### 9.6 结构体字段注释

结构体字段使用**行尾注释**（简短场景）或**行上注释**（需要多行解释）：

```go
type WithdrawalRecord struct {
    ID             int64           `json:"id"`
    IdempotencyKey string          `json:"idempotency_key"` // 幂等键，由上游业务生成
    Status         WithdrawStatus  `json:"status"`          // 提现状态，见 WithdrawStatus 枚举
    // RiskResult 保存风控引擎的完整响应。
    // 仅在 status >= StatusRiskChecked 时有值，下游审计依赖此字段。
    RiskResult     json.RawMessage `json:"risk_result"`
}
```

### 9.7 接口注释

接口注释应描述**契约**而非实现，说明调用方预期和实现方义务：

```go
// OrderRepository 定义订单持久化契约。
// 实现方必须保证同一 idempotency_key 的写入幂等。
// 所有方法必须支持 context 取消，超时时返回 context.DeadlineExceeded。
type OrderRepository interface {
    // Create 写入订单并返回完整实体。若幂等键冲突，返回已有记录而非 error。
    Create(ctx context.Context, order *Order) (*Order, error)

    // GetByID 根据主键查询，未找到时返回 ErrOrderNotFound。
    GetByID(ctx context.Context, id int64) (*Order, error)
}
```

### 9.8 禁止事项

| 禁止行为 | 示例  |
|------|-----|
| 复述代码的废话注释 | `// 设置 name 为 "foo"` → `name = "foo"` |
| 注释掉的代码提交 | `// order.Cancel()` 残留在主干 |
| 用注释替代命名 | 不要写 `var a int // 用户ID`，直接用 `userID` |
| 分隔线/装饰性注释 | `// =============================` |
| 日志式注释 | `// 2024-03-01 张三添加了此函数` — 用 Git 追踪 |


---

## 10. 性能要求

* 数据库查询必须设置 context timeout
* HTTP 客户端必须设置超时与连接池上限
* 禁止在循环内做数据库查询（N+1 问题）
* 大批量数据处理使用 stream 或分批，禁止一次性加载到内存
* 使用 `sync.Pool` 复用高频临时对象


---

## 11. 测试要求

* 单元测试文件与源文件同目录：`service.go` → `service_test.go`
* 表驱动测试为首选模式
* Mock 使用 `go.uber.org/mock` (mockgen) 或 `testify/mock`
* 新代码单测覆盖率 ≥ 80%，核心交易链路 ≥ 90%

```go
func TestOrderService_CreateOrder(t *testing.T) {
    tests := []struct {
        name    string
        input   CreateOrderReq
        wantErr bool
    }{
        {"valid order", CreateOrderReq{...}, false},
        {"missing amount", CreateOrderReq{...}, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // arrange, act, assert
        })
    }
}
```