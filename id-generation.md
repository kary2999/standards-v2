---
title: "订单号 / 分布式 ID 生成规范"
version: "1.0.0"
last_modified: "2026-06-12"
source: "技术规范 / ID 生成规范"
---

# 订单号 / 分布式 ID 生成规范

> 版本：V1.0.0 | 状态：生效 | 适用范围：所有 Go 微服务的订单号及分布式主键
>
> 本规范定义**新表 / 新服务**的订单号与分布式主键生成方式，取代旧的
> `BIGINT GENERATED ALWAYS AS IDENTITY` 自增主键约定（存量表迁移见 §5）。

---

## 1. 总则

订单号及所有需要跨服务、跨库唯一的主键 ID，**一律使用 UUIDv7**。

UUIDv7（RFC 9562）前 48 位是 Unix 毫秒时间戳，天然**时间有序**，兼具全局唯一
与索引友好，是当前团队的唯一发号标准。

**禁止**以下发号方式（新代码一经发现，CI / Review 直接打回）：

- ❌ 数据库自增（`BIGINT IDENTITY` / `AUTO_INCREMENT`）—— 跨库不唯一、暴露业务量、迁移困难
- ❌ 雪花算法（Snowflake）及各类自研发号器 —— 依赖时钟与 worker-id 分配，时钟回拨风险，运维成本高
- ❌ `order_no` 顺序号 / 日期拼计数器 —— 可预测、可枚举、并发下需分布式锁
- ❌ UUIDv4（纯随机）—— 无序，作 B-Tree 主键导致页分裂与写放大

---

## 2. Go 实现

统一用 `github.com/google/uuid`（v1.6.0+ 提供 `NewV7`），并**封装到 `mask-go-common-lib`**，
业务代码禁止直接调用底层库自己拼。

```go
// common-lib：idgen 包（唯一入口）
package idgen

import "github.com/google/uuid"

// NewOrderID 生成订单号 / 分布式主键（UUIDv7，时间有序）。
func NewOrderID() (uuid.UUID, error) {
    return uuid.NewV7()
}

// MustNewOrderID 仅用于初始化 / 测试；业务路径必须用 NewOrderID 处理 error。
func MustNewOrderID() uuid.UUID {
    id, err := uuid.NewV7()
    if err != nil {
        panic(err) // 熵源不可用属于不可恢复错误
    }
    return id
}
```

业务侧：

```go
id, err := idgen.NewOrderID()
if err != nil {
    return nil, fmt.Errorf("gen order id: %w", err)
}
order.OrderID = id
```

- 禁止在业务代码里出现 `uuid.NewV7()` / `uuid.New()` 裸调用，一律走 `idgen`
- 禁止把 UUID 转成 `int64` 或截断使用

---

## 3. 存储（PostgreSQL）

- 主键列类型用原生 **`uuid`**（非 `varchar`/`text`，省一半空间且比较更快）
- 主键定义：`order_id uuid PRIMARY KEY`
- 字段命名仍遵循 [field-naming.md](field-naming.md)：主体订单号叫 `order_id`，客户端幂等键 `client_order_id`，父单 `parent_order_id`
- UUIDv7 时间有序 → 顺序写入，B-Tree 友好，无需额外时间索引即可支撑按创建时间的范围扫描

```sql
CREATE TABLE trade_order (
    order_id        uuid PRIMARY KEY,          -- UUIDv7，由应用生成
    client_order_id varchar(64) NOT NULL,      -- 幂等键，客户端传入
    account_id      uuid NOT NULL,
    ...
    created_at      timestamptz NOT NULL DEFAULT now()
);
```

- ID 由**应用层生成**后写入，不使用 `gen_random_uuid()` 数据库默认值（数据库生成的是 UUIDv4，无序）

---

## 4. 对外展示与可读性

- UUIDv7 对外直接以标准 36 位字符串暴露（`018f...`），不额外编码
- 若产品需要「人类可读订单号」（客服报单等），另加**展示号**字段（如 `order_ref`），
  与主键 `order_id` 解耦；展示号不作主键、不参与关联
- 禁止把 UUID 暴露成可枚举的顺序值

---

## 5. 存量迁移

- 存量 `BIGINT` 自增主键的表**不强制**改造；新表 / 新服务一律 UUIDv7
- 需要改造的存量表：新增 `order_id uuid` 列并回填 → 双写过渡 → 切主键，按
  [database.md](database.md) §6 Migration 流程走 DBR 评审
- 跨新旧表关联期间，以 `client_order_id` 等业务幂等键做桥接

---

## 6. 检查清单

- [ ] 订单号 / 分布式主键用 `idgen.NewOrderID()`（UUIDv7），无裸 `uuid.*` 调用
- [ ] PG 主键列类型为 `uuid`，非 `varchar`
- [ ] 无数据库自增 / 雪花 / 顺序号发号
- [ ] ID 应用层生成，未用 `gen_random_uuid()` 默认值
- [ ] 需人类可读号时用独立 `order_ref`，不动主键
