# Team Standards v2 · 团队规范

> 技术栈：**Go 1.24 + Gin + MySQL 8.4 + Redis + Kafka**，前后端分开维护。
>
> 本仓库是 `go-skill-v2`（Team Standards App）同步的规范源，**独立于旧的 `kary2999/standards`**。旧 App 读旧仓库，不受本仓库改动影响。

## 文件清单

### 后端

| 文件 | 内容 |
|---|---|
| `go-style.md` | Go 编码风格、项目结构（handler → biz → data）、公共库速查、注释 / 日志 / Lint |
| `database.md` | **MySQL 8.4** 设计规范：命名、必备字段（`create_time` 毫秒 + `created_at`）、自增 `id` 主键 + UUIDv7 `oid` 业务 ID、无软删除、索引、分区、Migration、安全、运维 |
| `field-naming.md` | 全局字段命名词典（MySQL / Kafka / REST JSON 统一 snake_case） |
| `id-generation.md` | 两个 ID：`id` 自增（库内主键）+ `oid` UUIDv7（业务 ID / 订单号） |
| `redis.md` | Redis Key 命名、缓存策略、分布式锁、幂等、运维 |
| `kafka.md` | Kafka Topic 命名、消息信封、幂等消费、DLQ、Outbox |
| `error-codes.md` | 错误码体系 `{SERVICE}_{MODULE}_{NNN}` + `AppError` 用法 |
| `naming-logging.md` | 仓库 / Go / 中间件命名、链路追踪、日志字段 |
| `testing.md` | 测试规范 |

### 接口与前端

| 文件 | 内容 |
|---|---|
| `api-doc.md` | **REST + JSON 统一响应体**接口规范 + 接口文档模板 + 完整示例 |
| `frontend.md` | 前端通用约定（不限定框架）：前后端契约分工、请求封装、金额 / 时间处理、错误处理、安全、发布 |

### 流程与文档

| 文件 | 内容 |
|---|---|
| `tech-design-example.md` | 技术方案 10 章范例（订单服务） |
| `code-review.md` | Code Review 规范 |
| `ci-pipeline.md` | CI/CD 流水线 |
| `commit.md` | Git commit 规范 |
| `deployment-checklist.md` | 部署上线清单 |
| `feature-flags.md` | 特性开关 / 灰度发布 |
| `tixuebj-template-simple.md` | 提测文档模板 |
| `meeting-minutes.md` | 会议纪要模板 |
| `glossary.md` | 术语表 |
| `cursor-usage.md` | Cursor 使用约定 |
| `naming-visual.png` / `feature-flags-branch.png` | 速查图 |

## 与公共库的关系

规范中的 Go 用法对应公共库 `gin-go-common-lib`（`response` / `errors` / `naming` / `mq` / `idempotency` / `storage` …）。**规范与公共库不一致处已在各文档的“待确认”/“现状”提示中标注**，需要先改公共库再按规范落地，主要有：

* `util/idgen` 只有 UUIDv4，需新增 UUIDv7
* `mq.Consumer` 处理失败时不会重投该条消息（见 `kafka.md` §4.2）
* `response.BindJSONOrFail` 的校验错误字段名是 Go 字段名（见 `api-doc.md` §11）

## 版本

每个 `.md` 头部 frontmatter 标注 `version` + `last_modified`；仓库整体版本以最新 commit SHA 标识。当前为 **v2 草案**，待评审。
