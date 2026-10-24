---
title: "Kafka 使用规范"
version: "1.0.0-draft"
last_modified: "2026-10-22"
---
<!-- [skill: go-team-standards · Kafka · 命名规范] Kafka Topic 命名与消息规范 -->

# Kafka 使用规范

> 版本：V1.0.0-draft | 状态：草案 | 适用：Go + Gin + MySQL 8.4 + Kafka（公共库 `gin-go-common-lib/mq`，底层 `segmentio/kafka-go`）
>
> Kafka 的职责：**服务间异步事件、削峰、最终一致**。它不是事实源（MySQL 才是），也不用于请求-响应式调用（走 REST）。

---

## 1. Topic 命名

**项目内：** `{env}_{service}_{biz}_{action}`（公共库 `naming.TopicInProject`）

| 维度 | 规范 | 示例 |
|---|---|---|
| env | 白名单 `dev` / `test` / `staging` / `prod` | `prod` |
| service | 发布方服务短名 | `order`、`wallet`、`trade` |
| biz | 业务语义，小写下划线 | `pay_status`、`deposit` |
| action | 过去式动词 | `updated`、`confirmed`、`matched` |

示例：`prod_order_pay_status_updated`、`prod_wallet_deposit_confirmed`

**跨项目：** `{env}_{from}_2_{to}_{biz}`（`naming.TopicCrossProject`），例 `prod_pay_2_order_record`

**约束（`naming.ValidateTopic` 强制）：**

* 只允许**小写字母、数字、下划线**——**不得出现驼峰（`payStatus`）、`.`、`-`**
* 必须带环境前缀
* **禁止**把 `platform_id` / `symbol` 编入 Topic 名（放消息头或消息体）
* 构造 Topic 必须用 `naming` 包，禁止手拼字符串
* **消费者组：** `{systemID}_{purpose}`（`naming.ConsumerGroup`），例 `order_service_notify_send`
* 死信队列：`{原 topic}_dlq`

> 以 `naming` 包的校验规则为准。早期文档里出现过的 `<域>.<事件>.v1`（点分隔 + 版本号）风格**已废弃**，与公共库校验冲突。

---

## 2. 消息格式

### 2.1 统一信封

消息体统一为 JSON（UTF-8），字段 `snake_case`：

```json
{
  "event_id": "0194f3a2-7c1e-7b52-9d0a-3f6c2e8a1b44",
  "event_type": "order.pay_status_updated",
  "schema_version": 1,
  "occurred_time": 1761109000123,
  "platform_id": "p001",
  "payload": {
    "order_oid": "0194f3a2-7c1e-7b52-9d0a-3f6c2e8a1b45",
    "uid": "usr_88001",
    "pay_status": 2
  }
}
```

| 字段 | 说明 |
|---|---|
| `event_id` | 事件唯一 ID（UUIDv7，`idgen` 生成；落库时即 `outbox_events.oid`），**消费端幂等的依据**。载荷里引用业务实体一律用 `{实体}_oid`，不得携带自增 `id` |
| `event_type` | 事件类型，小写，便于路由和排查 |
| `schema_version` | 载荷结构版本，整数 |
| `occurred_time` | 事件发生时间，毫秒时间戳（UTC） |
| `platform_id` | 多租户标识 |
| `payload` | 业务数据；金额用**字符串**，ID 用 36 位 UUID 字符串 |

### 2.2 消息头

通过 `Producer.PublishWithHeaders` 传递：

| Header | 说明 |
|---|---|
| `trace_id` | 链路追踪 ID，与 HTTP 的 `X-Trace-Id` 一致 |
| `platform_id` | 多租户标识 |
| `idempotency_key` | 业务幂等键（通常等于 `event_id`） |

### 2.3 Schema 演进

* 载荷结构变更**只允许向后兼容**：新增可选字段；禁止删除字段、改字段类型或语义
* 破坏性变更必须新增 `action`/`biz` 创建新 Topic，旧 Topic 保留到所有消费者迁移完毕
* 载荷的 Go 结构体放在发布方服务的 `contract/` 包（或独立 Go Module）中，消费方引用同一份定义，**禁止消费方自己重新定义结构体**

### 2.4 分区键

* 需要**顺序**的事件（同一订单、同一账户）用业务主键作为消息 Key（`order_oid` / `account_oid`），保证同 Key 进同分区
* 全局顺序不保证；需要跨实体顺序的场景须重新设计
* 分区数一经设定只增不减，建议 ≥ 消费者实例数的 2 倍

---

## 3. 生产者

* 默认**同步发送**（`Async=false`），关键事件必须确认发送成功
* `Publish` 的 `ctx` 必须带超时（建议 3s）
* **“写库 + 发消息”不是一个事务**，不得先提交 MySQL 再 `Publish`（进程崩溃会丢消息）。必须用 §5 的 Outbox 模式
* 生产环境 Topic 由运维 / IDP **预先创建**并配置 ACL；公共库 Producer 默认 `AllowAutoTopicCreation: true`，仅 dev / test 使用，**生产配置须关闭**

---

## 4. 消费者

### 4.1 必须幂等

Kafka 至少一次投递，消息**一定会重复**（重平衡、重试、重发）。消费端必须：

1. 以 `event_id`（或 `idempotency_key`）去重
2. 去重记录与业务变更**在同一个 MySQL 事务内**提交：

```sql
-- 消费去重表：同一消费者组对同一事件只处理一次（纯技术表，无对外身份，故没有 oid）
CREATE TABLE `trade`.`consumed_events` (
    `id`              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '库内主键，自增',
    `consumer_group`  VARCHAR(128)    NOT NULL COMMENT '消费者组',
    `event_id`        CHAR(36)        NOT NULL COMMENT '事件 ID（UUIDv7 小写 36 位）',
    `create_time`     BIGINT UNSIGNED NOT NULL COMMENT '创建时间，毫秒时间戳（UTC）',
    `update_time`     BIGINT UNSIGNED NOT NULL COMMENT '更新时间，毫秒时间戳（UTC）',
    `created_at`      DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    `updated_at`      DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_consumed_events_group_event` (`consumer_group`, `event_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_bin COMMENT='消费去重表';
```

事务内先 `INSERT` 去重行：遇到 1062（唯一键冲突）说明已处理过，**直接视为成功**并提交 offset。

* Redis 可作为去重的快速路径，但**不能取代** MySQL 去重（见 [redis.md](redis.md) §6）
* 去重表按月清理 `create_time` 早于 30 天的数据（Kafka 保留期之外无需再去重）

### 4.2 重试与死信

> ⚠️ **公共库现状（已读源码核实，未连真实 Kafka 验证）**：`mq.Consumer.Start` 在 handler 返回 error 时只是 `continue`，**既不提交该条，也不会重新投递它**；而 kafka-go 的 offset 是水位线，后续消息提交成功后会**越过**这条失败消息，导致它被**静默丢弃**，并且没有任何日志。因此：
>
> **不得依赖“返回 error 让 Kafka 重试”。** 重试和死信必须在 handler 内部完成。

规范做法：

1. handler 内对**可重试错误**（DB 超时、下游 5xx）做进程内有限重试：最多 **3 次**，指数退避 1s / 5s / 30s
2. 仍失败，或属于**不可重试错误**（消息格式非法、业务规则拒绝）：把原消息写入死信 Topic `{原 topic}_dlq`，并**返回 nil**（表示本条已处置）
3. DLQ 必须接入告警，业务 Owner 24 小时内处理；DLQ 消息保留 ≥ 7 天
4. 业务 handler 内禁止 `panic`（启动期校验除外）；意外 panic 必须被 recover 并按“不可重试错误”走 DLQ

### 4.3 其它

* 消费者组 `GroupID` 遵循 `naming.ConsumerGroup`
* 起始位点：公共库未显式配置时默认 `FirstOffset`（新组从头读）；“不重放历史”的消费者须显式传 `LastOffset`（注意：公共库 `ConsumerConfig.StartOffset` 的注释写“默认 LastOffset”，与实际代码不符，以代码为准）
* handler 必须使用传入的 `ctx`，下游调用带超时
* 单条消息处理时间 > `SessionTimeout`（默认 30s）会触发重平衡，须拆分或异步
* 消费滞后（lag）接入监控，持续增长告警

---

## 5. Outbox：MySQL 与 Kafka 的一致性

**问题：** 业务写 MySQL 成功但发 Kafka 失败（或反过来），数据就不一致。

**做法：本地消息表（Outbox）**

1. 业务变更与 `outbox_events` 插入**在同一个 MySQL 事务**里提交
2. 独立的 relay 进程轮询 `outbox_events`，把 `event_status=待发送` 的记录发到 Kafka，成功后更新状态
3. 发送失败按退避重试；超过阈值转人工处理并告警

```sql
CREATE TABLE `trade`.`outbox_events` (
    `id`            BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '库内主键，自增',
    `oid`           CHAR(36)         NOT NULL COMMENT '业务 ID，UUIDv7 小写 36 位，同时作为信封里的 event_id',
    `topic`         VARCHAR(128)     NOT NULL COMMENT '目标 Topic',
    `msg_key`       VARCHAR(64)      NOT NULL COMMENT '消息 Key（保证同实体有序）',
    `payload`       JSON             NOT NULL COMMENT '消息体信封',
    `event_status`  TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态：1=待发送 2=已发送 3=发送失败',
    `retry_count`   INT UNSIGNED     NOT NULL DEFAULT 0 COMMENT '重试次数',
    `create_time`   BIGINT UNSIGNED  NOT NULL COMMENT '创建时间，毫秒时间戳（UTC）',
    `update_time`   BIGINT UNSIGNED  NOT NULL COMMENT '更新时间，毫秒时间戳（UTC）',
    `created_at`    DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    `updated_at`    DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_outbox_events_oid` (`oid`),
    KEY `idx_outbox_events_status_ctime` (`event_status`, `create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_bin COMMENT='本地消息表';
```

* relay 取数必须分批（≤ 100 条）、按 `create_time, id` 排序；多实例并行时用 `SELECT ... FOR UPDATE SKIP LOCKED` 或按 `msg_key` 哈希分片，保证同一 `msg_key` 的事件**串行发送**
* 已发送记录按 7 天保留后清理（分批 `DELETE`，见 [database.md](database.md) §4.4）
* relay 是“至少一次”发送，**消费端幂等是前提**（§4.1）

---

## 6. 运维

* Topic：副本数 ≥ 3，`min.insync.replicas=2`；生产者 `acks=all`
* 保留期默认 7 天；日志类 Topic 可缩短，资金类事件按审计要求延长
* Topic 创建 / 修改走运维流程，业务服务账号只授读 / 写权限，不授管理权限
* 监控：消费滞后、DLQ 堆积、Outbox 待发送积压、发送失败率

---

## 7. Go 侧落地约定

| 项 | 约定 |
|---|---|
| 生产者 | `mq.NewProducer(cfg)`；发送用 `PublishWithHeaders`，带 `trace_id` / `platform_id` / `idempotency_key` |
| 消费者 | `mq.NewConsumer(cfg, handler)` + `Start(ctx)`；handler 自己负责重试 / DLQ（§4.2） |
| 命名 | 一律 `naming.TopicInProject` / `TopicCrossProject` / `ConsumerGroup` |
| 超时 | 所有 `Publish` 带超时；handler 内下游调用带超时 |
| 日志 | 发送 / 消费失败用 `slog` 带 `trace_id`、`topic`、`event_id`；**消息体禁止整体入日志**（可能含敏感数据） |
| 指标 | label 用 `topic` / `result`，**禁用 `event_id` / `uid`** |

---

## 8. 检查清单

- [ ] Topic / 消费者组由 `naming` 包构造，小写下划线，带环境前缀
- [ ] 消息为统一信封，含 `event_id` / `schema_version` / `occurred_time`
- [ ] 写库 + 发消息用 Outbox，没有“提交后再发”
- [ ] 消费端在同一事务内写去重表
- [ ] handler 内有重试 + DLQ，不依赖“返回 error 重试”
- [ ] 生产环境已关闭自动建 Topic
- [ ] DLQ 与消费滞后已配告警

---

## 版本记录

| 版本 | 日期 | 摘要 |
|---|---|---|
| 1.0.0-draft | 2026-10-22 | 初稿：Topic 命名对齐 `naming` 包，统一消息信封，Outbox / 去重表（MySQL），消费重试与 DLQ，并标注公共库 `mq.Consumer` 的重试缺陷 |
