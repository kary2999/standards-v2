---
title: "术语表 — Glossary"
version: "1.0.0"
last_modified: "2026-04-26"
source: "团队原始（无更新）"
---

# 术语表 — Glossary

面向 10 年 PHP/Swoole 转 Go 的同学。只解释 **Go 生态、团队自建组件、云原生、可观测性** 相关术语。通用概念（HTTP、JSON、MySQL、Redis、SQL）不再解释。

---

## 1. Go 语言生态

| 术语 | 含义 | PHP/Swoole 类比 |
|---|---|---|
| **goroutine** | Go 运行时管理的轻量协程，`go func()` 即启动 | ≈ Swoole 协程，但更便宜、调度由 runtime 托管 |
| **channel** | goroutine 间通信的类型化管道，阻塞/非阻塞皆可 | ≈ Swoole Channel |
| **context.Context** | 贯穿调用链的上下文对象，携带 deadline / cancel / trace 等 | PHP 请求级全局变量的替代品（但显式传递） |
| **errgroup** | 一组 goroutine 协同等待 + 错误汇集（`golang.org/x/sync/errgroup`） | PHP `Swoole\Coroutine\WaitGroup` 升级版 |
| **defer** | 函数返回时按 LIFO 顺序执行，常用于释放资源 | 约等于 try/finally |
| **slog** | Go 1.21+ 标准库结构化日志 | ≈ monolog/psr-3 但是官方 |
| **decimal.Decimal** | `github.com/shopspring/decimal`，高精度十进制 | 避免 float64，交易必用 |
| **generic / 泛型** | Go 1.18+ 支持 `func Foo[T any](x T)` | PHP 无对应，类似 C++ template |

## 2. 构建 / 依赖 / 代码生成

| 术语 | 含义 |
|---|---|
| **go.mod / go.sum** | 模块声明 + 依赖锁定文件，等价于 `composer.json` + `composer.lock` |
| **Kratos** | B 站开源的 Go 微服务框架（本团队主力）；提供 HTTP + gRPC 统一传输层、middleware 链、配置、服务发现等 |
| **Wire** | Google 出品的编译期依赖注入；写 `wire.go` 声明 Provider，`wire_gen.go` 自动生成构造代码（不是运行时反射） |
| **Buf** | Protobuf 管理工具，做 lint / breaking change 检测 / 生成代码（替代裸 `protoc`） |
| **buf.yaml / buf.gen.yaml / buf.lock** | Buf 的模块声明 / 生成配置 / 依赖锁 |
| **protoc-gen-go / -grpc / -http** | Buf 调用的代码生成插件；最后一个生成 Kratos HTTP 路由绑定（gRPC + REST 双协议同源） |
| **ProviderSet** | Wire 里一组 Provider 的集合，可复用注入 |
| **goose / golang-migrate** | SQL 迁移工具，`up.sql` / `down.sql` 版本化；本团队两者都允许 |
| **asynq** | Redis 驱动的 Go 异步任务/cron 框架，本团队 `cron` 包基于它 |
| **GORM** | Go 最流行的 ORM（支持 MySQL / Postgres / ClickHouse） |

## 3. 团队自建组件（`mask-go-common-lib`）

| 包 | 职责 | 你会在什么时候用 |
|---|---|---|
| `logging` | 结构化 JSON 日志，自动从 ctx 提取 `trace_id` / `span_id` / `parent_span_id` | 每个服务必用，替代 `fmt.Println` |
| `tracing` | OTel 初始化 + W3C propagator + 采样策略（dev 全量 / prod 10%） | `main.go` 启动时调用一次 |
| `metrics` | Prometheus 工厂，`/metrics` 端点 | 业务指标暴露 |
| `middleware` | HTTP/gRPC 标准中间件链 + DB/Redis 连接池封装 | Server 初始化 |
| `httpclient` | 统一外呼 HTTP client，自动注入 span + 身份头 | 调用第三方接口 |
| `grpcclient` | 统一 gRPC client，自动 otelgrpc 拦截器 + metadata 透传 | 调用内部 gRPC 服务 |
| `config` | ConfigMap 友好配置加载，支持热加载 + Istio FQDN 拼接 | 读 `configs/dev/*.yaml` |
| `security` | 非 Istio 托管场景下的 mTLS credentials | 边缘场景 |
| `naming` | Kafka topic / Redis key **强制**命名规范助手 | 所有 MQ / 缓存操作 |
| `redisx` | Redis 读写封装，强制规范化 key：`Set(biz, module, id, val)` | 替代直接用 `go-redis` |
| `mq` | Kafka Producer / Consumer 封装，topic 命名强制；`SendBatch` 支持 | 替代直接用 `sarama`/`kafka-go` |
| `feature` | Feature Flag 框架，环境变量或 config 驱动（灰度默认关） | 新功能上线开关 |
| `alarm` | 异步去重告警，支持 日志 / MQ / Telegram 多渠道 | 业务异常兜底 |
| `errors` | 团队错误码 + 便捷构造函数（`ErrUnauthorized()` 等） | 业务返回错误 |
| `errno` | **错误码常量**（IDP 注册 → codegen 产物）；`xerror.New(errno.XxxCode)` 是标准写法 | 所有对外返回错误 |
| `header` | HTTP / gRPC header 常量 + 读写工具（`HeaderUserID` 等） | handler 层取身份 |
| `cron` | 基于 asynq 的定时任务，daemon + handler 注册 | 定时业务 |
| `router` | HTTP 路由注册接口（Kratos 适配） | 少数定制 |

**核心认知**：共工具函数已封装，**禁止绕过 common-lib 直接用底层库**（比如裸 `sarama`、裸 `go-redis`），否则命名规范和可观测性会被架空。

## 4. 可观测性

| 术语 | 含义 |
|---|---|
| **OpenTelemetry / OTel** | 厂商中立的可观测标准；定义 metrics / traces / logs 的数据模型和 SDK |
| **W3C Trace Context** | HTTP header `traceparent` 传递 `trace_id`/`span_id` 的标准 |
| **trace_id** | 一次端到端请求的全局 ID（32 位 hex），跨服务不变 |
| **span_id** | 一次操作的 ID（16 位 hex），每个 RPC / DB 调用都是一个 span |
| **parent_span_id** | 触发当前 span 的父操作 ID，用于还原调用树 |
| **Prometheus** | 拉模式指标系统，团队 `/metrics` 端点由它抓取 |
| **Loki** | Grafana 的日志聚合系统，结构化 JSON 日志天然亲和 |
| **Tempo** | Grafana 的分布式追踪存储，配合 OTel |
| **VictoriaMetrics** | 兼容 Prometheus 的高性能时序数据库 |
| **Grafana** | 统一展示面板（metrics + logs + traces） |
| **DeepFlow** | eBPF 驱动的应用拓扑 / 链路可视化工具 |

## 5. 云原生 / Kubernetes / 服务网格

| 术语 | 含义 |
|---|---|
| **K8s / Kubernetes** | 容器编排平台 |
| **Pod** | K8s 最小调度单元，1 个或多个容器共享网络 / 卷 |
| **Deployment** | 管理 Pod 副本和滚动升级的 K8s 资源 |
| **Service** | K8s 的虚拟 IP + DNS，暴露一组 Pod |
| **ConfigMap** | K8s 配置资源；挂载为文件后，应用可监听变更热加载 |
| **Secret** | K8s 敏感配置资源，base64 编码（**不是加密**，需 KMS/Vault 补强） |
| **ServiceFQDN** | `service.namespace.svc.cluster.local`，K8s 内部 DNS 全限定名 |
| **Istio** | 服务网格，通过 sidecar 自动注入 mTLS / 限流 / 重试 / 追踪 |
| **Envoy** | Istio 的数据平面代理 |
| **sidecar** | 与业务容器同 Pod 的辅助容器（通常是 Envoy） |
| **mTLS** | 双向 TLS，客户端与服务端互验证书；Istio 可自动托管 |
| **SCIM** | 企业 SSO 账号自动同步协议；Cursor Enterprise 用它对接 HR 系统 |
| **Harbor** | 企业级 Docker 镜像仓库 |
| **distroless** | Google 最小化容器基础镜像，只含应用 + 必要运行时，无 shell / 包管理器 |
| **Backstage** | Spotify 开源的 IDP 门户（内部开发者平台），本团队用作错误码 / 配置注册台 |
| **IDP** | Internal Developer Platform，内部开发者平台的统称 |
| **GitOps** | 用 Git 作为声明式配置的单一事实源，控制面板自动同步 |

## 6. CI / CD 与质量门禁

| 术语 | 含义 |
|---|---|
| **MR / PR** | Merge Request / Pull Request，代码评审入口 |
| **Pipeline** | CI 流水线；本团队阶段 `validate → build → test → scan → package → deploy` |
| **SAST** | Static Application Security Testing，静态代码安全扫描 |
| **commitlint** | 校验 commit message 是否符合 Conventional Commits |
| **husky** | Git hooks 管理工具（本地执行 commitlint 等） |
| **golangci-lint** | Go 社区聚合 lint 工具；团队 `.golangci.yml` 统一配置 |
| **Protected Branch** | 受保护分支（`main`），禁直 push，必须走 MR 合并 |

## 7. 安全 / 合规

| 术语 | 含义 |
|---|---|
| **KMS** | Key Management Service，密钥托管 |
| **HSM** | Hardware Security Module，硬件级密钥保护 |
| **2FA** | 二次验证（TOTP / WebAuthn 等） |
| **ZDR** | Zero Data Retention，零数据保留协议；Cursor Enterprise 强制模型侧不留存代码 |
| **PII** | Personally Identifiable Information，个人身份信息 |
| **KYC** | Know Your Customer，身份认证 |
| **AML** | Anti-Money Laundering，反洗钱 |

## 8. 消息队列特有术语

| 术语 | 含义 |
|---|---|
| **Kafka Topic** | 消息通道；团队命名 `{env}_{service}_{entity}_{action}` |
| **Partition** | Topic 分区，并行单位；同一 key 进同一分区 |
| **Consumer Group** | 一组消费者共享订阅；partition 在组内唯一分配 |
| **Offset** | Consumer 在 partition 中的读取位置 |
| **Pulsar** | Apache 另一个 MQ，sample-service 中有 demo（团队并非主力，但 common-lib 有兼容） |

## 9. 常见命令 / 工具

| 命令 | 用途 |
|---|---|
| `go mod tidy` | 清理 + 补全依赖，≈ `composer update` |
| `go vet` | 官方静态检查 |
| `go test -race -cover ./...` | 跑测试 + 竞态检测 + 覆盖率 |
| `buf generate` | 从 `.proto` 生成 Go 代码 |
| `wire` | 生成 `wire_gen.go` 依赖注入代码 |
| `goose up / down` | 数据库迁移前进 / 回滚 |
| `dlv` | Delve，Go 官方调试器 |
| `pprof` | Go 性能分析（CPU/内存/goroutine profile） |

---

## 附：从 PHP/Swoole 到 Go 的心智转换

1. **进程生命周期**：PHP 请求结束即销毁全局状态；Go 是常驻进程，连接池 / 全局状态贯穿整个进程寿命 → 必须显式管理。
2. **并发模型**：Swoole 协程 ≈ goroutine，但 Go 要求**显式传递 context** 做取消 / 超时 / trace 串联。
3. **错误处理**：PHP 抛异常；Go **返回 error**，必须每次显式检查。用 `fmt.Errorf("...: %w", err)` 保留错误链。
4. **依赖注入**：PHP 常用运行时容器（Laravel IoC）；Go 用 Wire **编译期**生成构造函数，无反射、启动快。
5. **ORM 习惯**：PHP Laravel/Doctrine 很"魔法"；GORM 相对克制，复杂查询直接写 SQL 反而更清晰。
6. **包管理心智**：不是 Composer，而是 `go mod` + `go.sum`；**目录即包名**。
7. **部署单位**：不是 PHP-FPM + Nginx，而是**单个二进制 + distroless 镜像**。Dockerfile 多阶段构建。
