---
title: "Redis 使用规范"
version: "1.0.0-draft"
last_modified: "2026-10-22"
---
<!-- [skill: go-team-standards · Redis · 命名规范] Redis 使用与 Key 命名规范 -->

# Redis 使用规范

> 版本：V1.0.0-draft | 状态：草案 | 适用：Go + Gin + MySQL 8.4 + Redis（go-redis v9）
>
> 与 MySQL 的分工：**MySQL 是唯一事实源，Redis 只是缓存 / 计数 / 锁 / 幂等**。任何“Redis 里有、MySQL 里没有就无法恢复”的数据（余额、订单状态、成交）都不得只存 Redis。

---

## 1. Key 命名

**格式：** `{业务域}:{模块}:{唯一标识}[:{子键}]`

| 维度 | 规范 | 示例 |
|---|---|---|
| 业务域 | 与 MySQL 库名对齐 | `wallet` / `trade` / `auth` / `market` / `risk` |
| 模块 | 域内二级功能 | `balance` / `order` / `session` / `limit` |
| 唯一标识 | 业务主键；多租户须带 `platform_id` | `p001:usr_88001` |
| 子键 | 可选细分 | `btc` / `daily` |

示例：`wallet:balance:p001:usr_88001:btc`、`risk:limit:p001:usr_88001:withdraw_daily`

**约束（公共库 `naming.ValidateRedisKey` 强制）：**

* 只允许小写字母、数字、`:`、`_`、`-`，**不得有大写、空格、中文**
* 至少 3 段（`{业务域}:{模块}:{标识}`）
* 构造 Key 必须用 `naming.RedisKey(biz, module, id)`，禁止在业务代码里手拼字符串
* 唯一标识里的 UUID 用 **36 位小写字符串**（`BINARY(16)` 只用于 MySQL 存储，不进 Key）
* 所有 Key 必须带 TTL；确需永久的 Key 须在设计文档中说明并经评审
* Key 总长建议 ≤ 128 字节

---

## 2. 缓存策略

### 2.1 Cache-Aside（默认模式）

* **读**：先读 Redis，未命中读 MySQL，回填 Redis（带 TTL）
* **写**：**先写 MySQL，再删 Redis**；不要“更新缓存”，更新容易与库产生顺序错乱
* 删除缓存失败必须重试（本地重试 3 次；仍失败则发 Kafka 事件异步补删，见 [kafka.md](kafka.md)）
* 强一致场景（余额、订单状态、对账）**不走缓存，直接读主库**

### 2.2 三类经典问题

| 问题 | 对策 |
|---|---|
| 缓存穿透（查不存在的数据） | 缓存空值，TTL 短（30–60s）；参数先校验合法性 |
| 缓存击穿（热点 Key 过期瞬间） | 同进程用公共库 `cache/singleflight` 合并回源；跨进程用互斥锁回源（§5） |
| 缓存雪崩（大批 Key 同时过期） | TTL 加随机抖动：`ttl = base + rand(0, base*0.2)` |

### 2.3 TTL 建议

| 数据 | TTL |
|---|---|
| 行情、配置类缓存 | 1–10 分钟 |
| 用户资料类缓存 | 10–30 分钟 |
| 登录会话 / Token 黑名单 | 与 Token 有效期一致 |
| 幂等键 | 24 小时（按业务重试窗口调整） |
| 空值缓存 | 30–60 秒 |

---

## 3. 数据类型与大小限制

| 限制 | 阈值 |
|---|---|
| 单 Key value | ≤ **10 KB**（超出须拆分或改存储） |
| Hash / List / Set / ZSet 元素数 | ≤ **5000** |
| 单 Key QPS | ≤ **5000**（超出视为热 Key，需本地缓存 + 多副本） |

* 大 Key 每周扫描一次，超限 Key 由 Owner 7 天内整改
* 禁止在单个 Key 存整张表 / 大集合，改用 Hash 分片（`{key}:{bucket}`）
* 序列化统一 JSON（便于排查）；高频大对象如需更紧凑的编码（如 MessagePack），须在设计文档说明
* **金额以字符串存储**（`decimal.Decimal.String()`），禁止 `float64` 序列化

---

## 4. 命令使用约束

* **禁止**在生产使用 `KEYS`、`FLUSHALL`、`FLUSHDB`、`DEBUG`；遍历用 `SCAN`（带 `COUNT`）
* 禁止对大集合执行 `SMEMBERS` / `HGETALL` / `LRANGE 0 -1`；改用 `SSCAN` / `HSCAN` 或分页
* 批量操作用 Pipeline，单批 ≤ 100 条命令
* 多 Key 原子操作用 Lua 脚本；Cluster 模式下 Lua 涉及的所有 Key 必须落在同一 Slot（用 `{hash_tag}`）
* 禁止长时间阻塞命令；禁止在 Lua 里做重计算

---

## 5. 分布式锁

**Key：** `lock:{业务域}:{资源类型}:{资源ID}`，例 `lock:wallet:withdrawal:usr_88001`

* 加锁：`SET key token NX PX <ttl>`，`token` 为随机值（含服务名、实例、request_id）
* 释放：**Lua 脚本校验 token 后再 `DEL`**，禁止直接 `DEL`
* TTL 默认 10s，须大于业务执行时间；禁止用作长事务锁（> 30s），应改异步任务 + 状态机
* 获取失败不得无限自旋：最多重试 3 次，指数退避，然后返回业务错误
* **锁只用来削峰和防重复提交，不是资金正确性的依据。** 余额扣减、订单状态流转的正确性必须由 MySQL 乐观锁 / 唯一键保证（见 [database.md](database.md) §4.8）。因为 Redis 锁在主从切换、GC 停顿、TTL 过期时都可能失效

---

## 6. 幂等

* 公共库提供 `idempotency.Middleware(store, ttl)`（Gin 中间件）与 `idempotency.NewRedisStore(rdb)`，底层是 `SET NX`
* 客户端通过请求头 `X-Idempotency-Key` 传入（见 [api-doc.md](api-doc.md)）；重复请求中间件返回 HTTP 409 + `COMMON_COMMON_301`
* 中间件在 **Redis 异常时放行**（fail-open，避免基础设施故障拖垮业务），且未传该头时不拦截。因此 Redis 幂等只是**第一道**去重，**最终**去重必须靠 MySQL 唯一键（`idempotency_key` / `client_order_id`），避免 Redis 故障、重启或键过期后重复入账
* Key：`{biz}:idem:{idempotency_key}`

---

## 7. Pub/Sub 与 Stream

* Pub/Sub 只用于**无持久化要求**的即时广播（配置热加载、行情推送），订阅端必须容忍消息丢失
* 需要持久化 / 重放 / 至少一次投递的场景一律使用 Kafka，**禁止用 Redis Pub/Sub 替代**
* 格式：`pubsub:{env}:{业务域}:{事件}`，例 `pubsub:prod:market:symbol_update`

---

## 8. 实例与运维

* **缓存实例**与**锁 / 计数 / 幂等实例**分开部署：缓存实例 `maxmemory-policy allkeys-lru`；锁 / 幂等实例用 `noeviction`，避免关键 Key 被淘汰
* 生产使用 Redis Cluster 或哨兵，禁止单点
* 持久化：AOF `everysec`；Redis 仍只当缓存，持久化是为了加快重启预热，不是为了防丢数据
* 内存使用率 > 70% 告警；单 Key 大小、慢日志（`slowlog`）纳入监控
* 密码、TLS 配置从配置中心 / 环境变量读取，禁止硬编码

---

## 9. Go 侧落地约定

| 项 | 约定 |
|---|---|
| 客户端 | 公共库 `gin-go-common-lib/storage/redis`：`redis.New(cfg)` 返回 go-redis `UniversalClient`（单机 / 哨兵 / Cluster 统一） |
| 超时 | 所有调用必须传 `ctx` 且带超时（读 100–300ms，写 300ms–1s），禁止 `context.Background()` 直调 |
| 未命中 | `redis.Nil` 表示未命中，**不是错误**，必须单独分支处理；其他 error 才是故障 |
| 降级 | Redis 故障时：读请求降级读 MySQL（须有限流保护 MySQL）；写请求不得因删除缓存失败而返回失败 |
| Key 构造 | 一律 `naming.RedisKey`，禁止手拼 |
| 日志 | 慢命令、错误用 `slog` 带 `trace_id`；Key 中的 uid 可记录，**value 内容禁止入日志** |
| 指标 | Prometheus label 用 `cmd` / `result`，**禁用 Key、uid 作 label**（基数爆炸） |

---

## 10. 检查清单

- [ ] Key 通过 `naming.RedisKey` 构造，全小写且 ≥ 3 段
- [ ] 所有 Key 有 TTL，且加了随机抖动
- [ ] 无 `KEYS` / `HGETALL` 大集合 / 无 TTL 的锁
- [ ] 缓存更新遵循“先写库后删缓存”，删除失败有重试 / 补偿
- [ ] 资金正确性不依赖 Redis 锁
- [ ] 幂等有 MySQL 唯一键兜底
- [ ] 所有调用带超时，`redis.Nil` 单独处理
- [ ] 大 Key / 热 Key 已评估

---

## 版本记录

| 版本 | 日期 | 摘要 |
|---|---|---|
| 1.0.0-draft | 2026-10-22 | 初稿：Key 命名、缓存策略、分布式锁、幂等、运维，对齐 `gin-go-common-lib` 的 `naming` / `idempotency` / `storage/redis` |
