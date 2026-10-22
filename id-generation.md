---
title: "订单号 / 分布式 ID 生成规范"
version: "2.0.0-draft"
last_modified: "2026-10-22"
source: "技术规范 / ID 生成规范"
---

# 订单号 / 分布式 ID 生成规范

> 版本：V2.0.0-draft | 状态：**草案，待评审** | 适用范围：所有 Go 微服务的表主键与业务 ID
>
> **v2.0.0 变更**：每张业务表同时有两个 ID —— `id`（`BIGINT UNSIGNED AUTO_INCREMENT`，库内物理主键）和 `oid`（UUIDv7，应用层生成的业务 ID）。此前草案“只用 UUIDv7 `BINARY(16)` 作主键、禁止自增”作废。

---

## 1. 总则：两个 ID，分工明确

| 字段 | 类型 | 谁生成 | 角色 |
|---|---|---|---|
| `id` | `BIGINT UNSIGNED NOT NULL AUTO_INCREMENT` | **数据库** | **库内物理主键**（聚簇索引）。只在本表内部使用 |
| `oid` | `CHAR(36) NOT NULL`，带唯一索引 | **应用层**（UUIDv7，写入前已知） | **业务 ID**。订单号、对外展示、跨表 / 跨服务 / 跨库引用、API、Kafka、Redis、日志，都用它 |

一句话：**`id` 只给数据库自己用，`oid` 才是业务身份。**

### 1.1 `id`（自增主键）的使用边界

允许：
* 作为聚簇主键，使二级索引保持窄（每条二级索引记录只多带 8 字节）
* 同一张表内的稳定排序 / 游标 tie-breaker（`ORDER BY create_time DESC, id DESC`）
* 运维排查、`log_*` 表按 `id` 增量扫描

禁止：
* ❌ 出现在 API 响应、Kafka 消息、Redis Key、前端页面里（暴露业务量，且可被枚举）
* ❌ 被其他表 / 其他服务引用为逻辑外键（引用一律用 `oid`）
* ❌ 当作业务编号、订单号、幂等键、时间使用
* ❌ 业务逻辑读取 `LAST_INSERT_ID()` 或依赖“插入后由数据库返回的 `id`”——业务需要的标识是插入前就已生成的 `oid`
* ❌ 假设连续：回滚、批量插入、失败重试都会产生空洞，**不得**用 `max(id)` 估行数、不得用 `id` 差值判断丢失
* ❌ 跨库 / 跨分片唯一：分库分表后各片 `id` 会重复，跨片身份只认 `oid`

### 1.2 `oid`（UUIDv7）的规则

* 由应用层通过 common-lib 的 `idgen` 生成；禁止业务代码裸调 `uuid.NewV7()` / `uuid.New()`
* 存储为 **36 位小写带连字符字符串**，类型 `CHAR(36)`，排序规则沿用库默认 `utf8mb4_0900_bin`（区分大小写，所以**必须小写归一**后再写入、再查询）
* 每表必须有 `UNIQUE KEY uk_{表}_oid (oid)`；若表按 `platform_id` 做租户隔离，`oid` 仍然全局唯一，不与 `platform_id` 组合
* 引用其他实体时，字段命名 `{实体单数}_oid`（`order_oid`、`account_oid`），类型同样 `CHAR(36)`；**禁止**用 `{实体}_id` 存 `oid`（会与对方表的自增 `id` 混淆）
* API / Kafka 的 JSON 字段名与数据库一致：实体自己的是 `oid`，引用的是 `{实体}_oid`，无需映射
* **禁止** UUIDv4（无序，唯一索引页分裂与写放大）、UUIDv1（含 MAC 地址）；禁止把 `oid` 转 `int64` 或截断

### 1.3 禁止的发号方式

* ❌ 雪花算法（Snowflake）及各类自研发号器：依赖时钟与 worker-id，时钟回拨或配置重复会静默发出重复 ID
* ❌ `order_no` 顺序号 / 日期拼计数器：可预测、可枚举、并发下需分布式锁
* ❌ 用 `id`（自增）充当业务 ID 或订单号

---

## 2. 为什么是“自增 `id` + UUIDv7 `oid`”

InnoDB 二级索引的每条记录都带着主键值，所以主键越窄，所有二级索引越小：

| 方案 | 每条二级索引多带 |
|---|---|
| 主键 `BIGINT`（本规范 `id`） | 8 字节 |
| 主键 `CHAR(36)` | 36 字节 |

* 主键选自增 `BIGINT`：追加写入、聚簇索引紧凑、二级索引最窄
* 业务 ID 另设 `oid`：全局唯一、不可枚举、写入前已知（便于幂等、事件、重试），不依赖数据库
* `oid` 只在 `uk_*_oid` 这一个唯一索引里出现 36 字节（该索引每条记录还带 8 字节 `id`），**不会**被复制进其他二级索引，因此 `CHAR(36)` 的空间代价可接受，换来直接可读、ORM 零封装
* 相比单用 UUID 主键：二级索引更小；相比单用自增：对外不暴露、跨库不冲突

代价（评审时须知道）：

* 每表多一个唯一索引 `uk_*_oid`，写入多一次索引维护；按 `oid` 查询要先查唯一索引再回表（两次 B+ 树查找）
* `oid` 为什么是 `CHAR(36)` 且**不得改成** `BINARY(16)`：业务要求 `oid` 肉眼可读（查库、日志、工单直接复制粘贴），可读性优先于省空间

---

## 3. Go 实现

统一用 `github.com/google/uuid`（v1.6.0+ 提供 `NewV7`），并**封装到公共库 `gin-go-common-lib/util/idgen`**：

```go
// common-lib：idgen 包（唯一入口）
package idgen

import "github.com/google/uuid"

// NewOID 生成业务 ID（UUIDv7，时间有序）。
func NewOID() (uuid.UUID, error) {
    return uuid.NewV7()
}

// NewOIDString 生成 36 位小写带连字符字符串，直接用于 CHAR(36) 列。
func NewOIDString() (string, error) {
    u, err := uuid.NewV7()
    if err != nil {
        return "", err
    }
    return u.String(), nil // google/uuid 的 String() 恒为小写
}
```

> ⚠️ **公共库现状（已读代码核实）**：`gin-go-common-lib/util/idgen` 目前只有 `NewUUID()` / `NewUUIDWithHyphen()`，**都是 UUIDv4**，没有上面的函数。落地前须先在公共库新增 UUIDv7 生成函数，业务代码才能按本规范使用。

模型：

```go
type Order struct {
    ID         uint64 `gorm:"column:id;primaryKey;autoIncrement"`   // 库内主键，不进 DTO
    OID        string `gorm:"column:oid"`                           // 业务 ID，写入前由 idgen 生成
    // ...
    CreateTime int64  `gorm:"column:create_time;autoCreateTime:milli"`
    UpdateTime int64  `gorm:"column:update_time;autoUpdateTime:milli"`
}
```

* 创建时先 `oid := idgen.NewOIDString()`，再 INSERT；之后的事件、响应、日志都用 `oid`，不读回 `id`
* DTO / 事件 / 缓存 Key 里**不要**包含 `ID`（自增）字段
* **内存里大量暂存订单时**，ID 用 `uuid.UUID`（`[16]byte`），不要存 36 字符 `string`：实测（Go 1.24，100 万个）`string` 每个约 **64 字节**、`[16]byte` **16 字节**、`int64` **8 字节**；`[16]byte` 可直接当 map 的 key、直接 `==` 比较，边界处再 `.String()`。从库里读出的 `oid` 字符串用 `uuid.Parse` 转换

---

## 4. 撞号与幂等

* `oid` 撞号概率极低，且有 `uk_*_oid` 兜底：INSERT 报 MySQL 错误 `1062` 时，**看错误信息里的索引名**——命中 `uk_{表}_oid` 才重新生成 `oid` 重试（最多 3 次）；命中业务唯一键（`uk_*_client_order` 等）是幂等重复，按幂等语义处理，**不得**换 `oid` 重试
* 业务幂等靠 `client_order_id` / `idempotency_key` 等业务唯一键，不靠 `oid` 或 `id`

---

## 5. 对外展示与可读性

* `oid` 对外直接以标准 36 位字符串暴露（`018f...`），不额外编码
* 若产品需要「人类可读订单号」（客服报单等），另加**展示号**字段（如 `order_ref`），与 `oid` 解耦；展示号不作主键、不参与关联
* 禁止把 `id`（自增）暴露成订单号

---

## 6. 存量迁移

* 存量只有自增 `id` 的表：新增 `oid CHAR(36) NOT NULL DEFAULT ''` 列 → 分批回填 UUIDv7（回填脚本见 [database.md](database.md) §6.2 / §7.2 分批规则）→ 加唯一索引 → 应用改为用 `oid` 对外 → 最后把 `DEFAULT ''` 去掉。大表走 gh-ost，流程按 [database.md](database.md) §6 走 DBR 评审
* 回填期间，新旧代码并存时以 `client_order_id` 等业务幂等键做桥接

---

## 7. 检查清单

- [ ] 每张业务表同时有 `id`（`BIGINT UNSIGNED AUTO_INCREMENT`，主键）和 `oid`（`CHAR(36)`，带 `uk_*_oid`）
- [ ] `oid` 由 `idgen` 生成 UUIDv7、小写，无裸 `uuid.*` 调用
- [ ] 跨表 / 跨服务引用字段为 `{实体}_oid`，类型 `CHAR(36)`；没有用 `id` 做逻辑外键
- [ ] API / Kafka / Redis Key / 日志里没有自增 `id`
- [ ] 业务代码不读 `LAST_INSERT_ID()`，不依赖 `id` 连续
- [ ] 无雪花 / 顺序号 / UUIDv4
- [ ] 1062 重试按索引名区分 `oid` 撞号与业务幂等重复
- [ ] 公共库 `util/idgen` 已提供 UUIDv7 生成函数（见 §3 说明）
