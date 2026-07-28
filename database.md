---
title: "数据库设计规范"
version: "1.2.0"
last_modified: "2026-05-08"
source: "技术规范.2026.05.08 / 数据库设计规范.md"
---

# 数据库设计规范

# 数据库设计与变更规范

> 版本：V1.2.0 | 状态：生效 | 适用范围：PostgreSQL
>
> **v1.2.0 变更**：新增 §2 环境基线（版本/扩展/服务器配置/连接池）、§5 分区与分片、§9 运维规范；§4 设计规范补充主键选型（§4.6）、乐观锁（§4.7）、审计表标准字段（§4.8）、CREATE TABLE 模板（§4.9）；§6 Migration 补充 DBR Checklist（§6.5）与压测要求（§6.6）；§7 查询规范补充事务与锁（§7.6）、慢查询监控（§7.7）；§8 数据安全补充数据脱敏（§8.5）；§10 Kafka 补充 Schema 注册与 DLQ；§11 Redis 补充大 Key / 分布式锁 / Pub/Sub 规范。章节重新编号（原 §2–§8 顺延）。
>
> **v1.1.0 变更**：新增 §4.5/§4.4 加密货币场景豁免条款；多租户字段宽度强制；Kafka 事件命名 / Redis Key 命名章节。


---

## 1. 总则

数据库是交易所系统的核心资产。所有 Schema 变更必须经过评审、版本化管理，并通过 CI Pipeline 自动执行。禁止任何人手动在生产环境执行 DDL。


---

## 2. 环境基线

### 2.1 数据库版本

* 生产环境使用 **PostgreSQL 18.x**（LTS 策略：主版本跟进 n-1，次版本季度滚动升级）
* 禁止在生产使用 Beta / RC 版本
* 升级窗口需提前一周公告，并完成灰度副本验证

### 2.2 扩展清单

仅允许使用以下扩展，新增需经 DBA 评审：

| 扩展  | 用途  |
|-----|-----|
| `pg_stat_statements` | SQL 性能统计（强制开启） |
| `pgcrypto` | 字段级加解密、随机数、哈希 |
| `uuid-ossp` / 内建 `gen_random_uuid()` | UUID 生成 |
| `pg_trgm` | 模糊匹配、ILIKE 加速 |
| `btree_gin` / `btree_gist` | 组合索引（JSONB + 标量字段） |
| `pgaudit` | 审计日志（生产强制） |
| `pg_repack` | 在线表重建（DBA 专用） |

### 2.3 服务器级配置

* `timezone = 'UTC'`（实例级强制）
* `lc_collate = 'C'`（索引性能最佳，业务排序由应用层处理）
* 服务端编码 `UTF8`，禁止使用其他 encoding
* `statement_timeout = 30s`（默认），长任务需会话级显式放宽
* `idle_in_transaction_session_timeout = 60s`，防止空闲事务占用连接
* `lock_timeout = 3s`，避免 DDL/写入长时间阻塞

### 2.4 连接池规范

* 所有服务必须通过 **PgBouncer** 接入数据库，禁止应用直连
* PgBouncer 模式固定为 `transaction`；使用 `SET LOCAL` 代替会话变量
* 单实例 `max_connections` 上限由 DBA 统一规划，应用层连接池计算公式： `pool_size = (服务副本数 × 单副本并发需求) + 预留缓冲`，单服务 pool 上限 **≤ 50**
* 禁止使用长事务（> 10s），必须拆分或改异步
* 禁止使用 `LISTEN/NOTIFY`（会绑定连接，与池化冲突）


---

## 3. 命名规范

### 3.1 通用规则

* 全部使用 `snake_case` 小写，禁止大写字母（PG 默认将未引用标识符转为小写，大写需引号引用，产生歧义）
* 名称应具有业务含义，禁止无意义缩写（公认缩写除外：`id`, `url`, `ip`, `uid`, `ts`）
* 对象名长度不超过 **63 字符**（PostgreSQL 内部限制）
* 禁止使用 SQL 保留字作为对象名（如 `order`, `user`, `group`, `date`, `value`）；可执行 `SELECT pg_get_keywords();` 查询完整保留字列表
* 禁止包含特殊字符（美元符号、空格、连字符），禁止使用非英文字母，禁止以 `pg_` 或 `_` 开头
* 禁止英文字母与数字无意义混杂（如 `order1`、`table2`）；数字只允许出现在有明确语义的后缀中（如 `address_line2`、`orders_2024q1`）
* 字段名不允许包含所在表名（如 `users` 表中禁止出现 `user_name`，应直接命名为 `name`）；关联字段作为外部引用时不受此限
* 禁止使用匈牙利命名法前缀（如 `tbl_`, `col_`）

### 3.2 Database 命名

* 格式：`{业务线}_{用途}` → `exchange_main`, `exchange_analytics`
* 分片库以 `_shard` 结尾 → `exchange_trade_shard`
* 禁止使用 `db1`、`test`、`database` 等无意义名称

### 3.3 Schema 命名

按业务模块划分 Schema，实现逻辑隔离：

| Schema | 用途  |
|--------|-----|
| `trade` | 撮合、订单、成交 |
| `wallet` | 资产、充提、余额 |
| `auth` | 用户、权限、KYC |
| `market` | 行情、K线、深度 |
| `risk` | 风控、反洗钱 |
| `audit` | 审计日志（只追加，禁止修改） |

* 禁止在 `public` schema 下创建业务表
* `search_path` 必须显式设置，禁止依赖默认值

### 3.4 表命名

#### 表类型前缀规范

每张表必须根据其职责选择对应前缀，前缀是表类型的唯一标识：

| 前缀  | 类型  | 命名格式 | 示例  |
|-----|-----|------|-----|
| 无前缀 | 核心业务表 | `{schema}.{实体复数}` | `trade.orders`, `wallet.balances` |
| `dim_` | 维度表 | `dim_{业务域}_{实体名}` | `dim_user_profile`, `dim_symbol_info` |
| `agg_` | 统计聚合表 | `agg_{业务域}_{指标}_{时间粒度}` | `agg_user_retention_weekly`, `agg_trade_volume_daily` |
| `cfg_` | 配置表 | `cfg_{业务/模块}_{配置含义}` | `cfg_system_params`, `cfg_fee_rules` |
| `log_` | 操作/事件日志表 | `log_{业务域}_{事件}` | `log_user_login`, `log_order_audit` |
| `hist_` | 历史快照表 | `hist_{业务域}_{实体}` | `hist_account_balance`, `hist_order_state` |
| `rel_` | 多对多关系表 | `rel_{关联A}_{关联B}`（按字母序） | `rel_user_role`, `rel_order_tag` |
| `tmp_` | 临时表 | `tmp_{业务域}_{业务}_{时间粒度}` | `tmp_user_migration_2026`, `tmp_settlement_batch` |
| `v_` | 视图  | `v_{描述}` | `v_active_orders`, `v_user_assets` |
| `mv_` | 物化视图 | `mv_{描述}` | `mv_daily_volume`, `mv_symbol_depth` |
| `{主表名}_ext` | 扩展表 | `{主表名}_ext` | `users_ext`, `orders_ext` |

#### 补充说明

* 核心业务表不加前缀，通过 Schema 隔离模块（见 §3.3），避免 `t_` 等冗余前缀
* `log_` / `hist_` 表只追加写入，禁止 UPDATE / DELETE，需在 Migration 中添加行级安全策略注释
* 扩展表（`_ext`）必须与主表 `id` 保持一对一关联，用于主表字段超出 30 个时的垂直拆分
* `tmp_` 临时表需在名称中体现时间特征，避免永久残留；超过 7 天未使用需清理
* 分区子表命名：`{父表名}_{分区特征}` → `orders_2024q1`, `orders_p0`

### 3.5 字段命名

#### 基础规则

* 主键 / 业务 ID：`id`（`BIGINT GENERATED ALWAYS AS IDENTITY` 或 UUID），直接作为业务 ID 对外暴露
* 关联字段（逻辑外键）：`{关联表单数}_id` → `order_id`（**禁止**使用数据库级 FOREIGN KEY 约束，关联关系由应用层维护）
* 禁止使用 PG 系统保留列名：`oid`, `xmin`, `xmax`, `cmin`, `cmax`, `ctid`
* 新增字段命名必须与同表现有字段风格保持一致
* 字段名不允许包含所在表名（如 `users` 表中直接命名 `name` 而非 `user_name`）

#### 字段语义后缀 / 前缀规范

| 场景  | 规则  | 示例  |
|-----|-----|-----|
| 时间戳（含时区） | `_at` 后缀 | `created_at`, `updated_at`, `deleted_at`, `executed_at` |
| 纯日期（不含时间） | `_date` 后缀 | `settlement_date`, `expire_date` |
| 金额/价格 | `_amount` 后缀 | `trade_amount`, `fee_amount`, `withdrawal_amount` |
| 数量/计数 | `_count` 后缀 | `order_count`, `retry_count`, `fill_count` |
| 过程流转状态 | `_status` 后缀 | `order_status`, `kyc_status`, `withdrawal_status` |
| 分类类型 | `_type` 后缀 | `order_type`, `account_type`, `fee_type` |
| 布尔判断 | `is_` / `has_` 前缀 | `is_active`, `is_deleted`, `has_verified` |
| 金额字段类型（法币 / 通用业务） | `DECIMAL(28,8)` | 禁止使用 `FLOAT` / `DOUBLE` |
| 金额字段类型（**加密货币场景**） | `NUMERIC(38,18)` | 见 §4.4 例外条款 |
| 费率字段（手续费率 / 利率 / 收益率） | `NUMERIC(10,8)` | 如 `maker_rate`、`taker_rate`、`apy` |

### 3.6 索引命名

* 主键约束：`{表}_pkey`（PG 默认规则，保持一致）
* 唯一索引：`uk_{表}_{字段}` → `uk_users_email`
* 普通索引：`idx_{表}_{字段}` → `idx_orders_uid`
* 组合索引：`idx_{表}_{字段1}_{字段2}`

### 3.7 约束命名

* Check 约束：`ck_{表}_{字段}` → `ck_orders_positive_amount`
* 唯一约束：`uq_{表}_{字段}` → `uq_users_email`

### 3.8 Role 命名

* 每个服务配置独立 Role，最小权限原则
* 格式：`{服务名}_{权限级别}` → `trade_service_read`, `trade_service_write`
* 禁止多个服务共用同一 Role
* 流复制专用 Role 固定命名为 `replication`

### 3.9 注释规范（COMMENT）

所有生产环境的表和字段**必须**添加 `COMMENT`：

```sql
COMMENT ON TABLE trade.orders IS '交易订单表，记录用户所有现货/合约订单';
COMMENT ON COLUMN trade.orders.status IS '订单状态：1=待撮合 2=部分成交 3=完全成交 4=已取消 5=已过期';
COMMENT ON COLUMN trade.orders.side IS '交易方向：1=买入(BID) 2=卖出(ASK)';
```

* 字段含义变更时必须同步更新 COMMENT，禁止注释与实际语义不符


---

## 4. 设计规范

### 4.1 必备字段

每张业务表必须包含以下字段：

```sql
CREATE TABLE trade.orders (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- ... 业务字段 ...
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ,           -- 软删除标记
    created_by    VARCHAR(64),
    updated_by    VARCHAR(64)
);
```

### 4.2 设计原则

* **禁止使用数据库级外键约束（FOREIGN KEY）**，所有关联关系由应用层负责维护，避免影响分库分表、高并发写入及 DDL 变更灵活性
* 除了TEXT类型字段和`deleted_at`， 业务字段默认不允许为NULL
* 只允许软删除：使用 `deleted_at` 标记，而非物理删除
* `deleted_at` 设置允许为 NULL
* 字段类型选择遵循 §4.4 字段类型约束
* 大文本 / JSON 字段评估是否需要，避免影响查询性能
* 单表字段不超过 **30 个**，超出考虑垂直拆分

### 4.3 索引策略

* 关联字段（如 `uid`、`order_id` 等逻辑外键字段）必须加索引
* WHERE / ORDER BY 高频字段加索引
* 单表索引数量不超过 **8 个**
* 组合索引遵循最左前缀原则
* 避免在高基数字段（如 `status` 只有几个值）上建普通索引
* **部分索引（Partial Index）**：高筛选度场景优先使用，例如 `CREATE INDEX idx_orders_open ON trade.orders(uid) WHERE status IN (1,2);`
* **表达式索引**：用于查询条件为函数结果（`LOWER(email)`、`date_trunc('day', created_at)`）时
* **覆盖索引（INCLUDE）**：读多写少场景可用 `INCLUDE` 减少回表，写多场景避免使用
* 索引创建 / 重建必须使用 `CONCURRENTLY`，避免锁表

### 4.4 字段类型约束

#### 时间字段

系统数据来源跨时区，所有时间字段必须遵循以下规范：

* 统一使用 `TIMESTAMPTZ(6)`，内部存储为 UTC，精度微秒级；存储开销与 `(3)` 完全一致（均为 8 bytes），无需按场景区分
* 所有时间值在应用层统一转换为 **UTC** 后写入，禁止写入本地时间或带偏移的时区时间
* API 对外输出时使用 **ISO 8601 格式**（如 `2024-01-01T00:00:00.000Z`），由应用层负责时区转换
* 禁止在数据库中存储时区字符串（如 `Asia/Tokyo`），时区信息属于用户偏好，存应用层

#### 金额 / 数量字段

* **默认（法币 / 通用业务场景）**：统一使用 `DECIMAL(28, 8)`，禁止使用 `FLOAT` / `DOUBLE`（浮点精度丢失在交易场景不可接受）
* 数量字段同样使用 `DECIMAL`，不得使用整型代替（除非单位已明确为最小不可分单位，如 satoshi）

##### 例外条款（加密货币场景）

为承载原生链上资产精度（BTC 8 位、ETH 18 位、Polkadot 10 位、Stacks 6 位、新兴高精度币 > 12 位），以下加密货币业务字段**必须**使用 `NUMERIC(38,18)`，不适用本节默认的 `DECIMAL(28,8)`：

| 字段类型 | 标准类型 | 举例  |
|------|------|-----|
| 链上资产数量 / 余额 / 冻结 | `NUMERIC(38,18)` | `balance`, `frozen`, `available`, `crypto_quantity` |
| 交易订单数量 / 成交数量 | `NUMERIC(38,18)` | `order_quantity`, `filled_qty`, `remaining_qty` |
| 充提数量 / 链上转账数量 | `NUMERIC(38,18)` | `deposit_amount`, `withdrawal_amount`, `tx_amount` |
| 矿工费（以原币计） | `NUMERIC(38,18)` | `miner_fee`, `gas_fee` |
| 期权 Premium / 权利金 | `NUMERIC(38,18)` | `premium`, `strike_price` |
| 云挖矿算力 / 产出 | `NUMERIC(38,18)` | `hashrate_th`, `daily_payout` |
| 手续费 / 利率 / 收益率 | `NUMERIC(10,8)` | `maker_rate`, `taker_rate`, `apy`, `interest_rate` |
| 汇率 / 价格 | `NUMERIC(38,18)` | `price`, `mark_price`, `index_price`, `usd_rate` |
| **商品/U 卡等法币等价金额** | **保留** `**DECIMAL(28,8)**` | `cash_amount`, `credit_limit`, `card_balance_usd` |

**约束：**

* 同一张表内"加密货币余额"与"法币等价"字段可并存（如 UCard 余额 + USD 估值），各自按其归属选择类型
* 类型变更需与 Tech Lead + DBA 双审批（§6.4）

#### 枚举 / 状态字段

* **禁止使用数据库 ENUM 类型**，统一使用 `SMALLINT` + 代码层常量映射
* 常量值从 `1` 开始，保留 `0` 表示未知 / 兜底状态

#### 字符串字段

* 固定格式字符串（如货币代码 `BTC`、语言代码）使用 `CHAR(n)`
* 变长字符串使用 `VARCHAR(n)`，明确最大长度
* **TEXT 类型**：允许使用，但仅限于无长度约束需求的字段（如备注、描述、日志内容）；作为查询条件的字段仍须使用 `VARCHAR(n)` 明确长度

#### JSON 字段

* PostgreSQL 仅允许使用 `**JSONB**`（二进制存储，支持 GIN 索引）；禁止使用 `JSON` 类型
* 仅用于**非核心交易数据**的扩展属性（如用户偏好配置、KYC 附加信息、审计日志扩展字段）
* 禁止将交易核心字段（金额、状态、时间）存入 JSONB
* 禁止以 JSONB 内部字段作为主要查询条件（应提取为独立字段加索引）
* 单字段大小建议不超过 **64KB**

### 4.5 多租户字段规范

涉及多业务线场景的业务表必须遵循以下强制字段类型：

| 字段  | 类型  | 可空  | 说明  |
|-----|-----|-----|-----|
| `platform_id` | `VARCHAR(32) NOT NULL` | ✗   | 平台/品牌 ID；全链路透传；作为 schema 分区/索引前缀 |
| `uid` | `VARCHAR(64) NOT NULL` | ✗   | 用户 ID；跨业务线全局唯一；禁止使用 `BIGINT` |
| `org_id` | `VARCHAR(64) NULL` | ✓   | 机构 ID；非机构账户为 NULL |

**强制约束：**

* 索引前缀必须至少包含 `(platform_id, ...)`，除非该表业务语义确定在单一 platform_id 内不跨表联查
* RLS（行级安全）策略必须以 `platform_id` 作为隔离维度：

  ```sql
  CREATE POLICY platform_isolation ON <table>
    USING (platform_id = current_setting('app.current_platform')::text);
  ```
* 全链路 Observability：每个 gRPC metadata / traceparent baggage / Kafka header / Redis key 必携带 `platform_id`

### 4.6 主键选型

| 场景  | 推荐类型 | 说明  |
|-----|------|-----|
| 单库单表 / 写入量中等 | `BIGINT GENERATED ALWAYS AS IDENTITY` | 顺序写入对 B-Tree 友好，空间小 |
| 分布式 / 分片 / 需对外暴露 | `UUID v7`（时间有序 UUID） | 保留时间顺序性，避免随机 UUID 的索引膨胀 |
| 高安全敏感（防枚举） | `UUID v4` | 不可预测，适合对外 ID |
| 大型日志 / 流水 | `BIGINT` + 分区键 | 配合分区表，避免单表过大 |

**禁止：**

* 使用 `SERIAL` / `BIGSERIAL`（老语法，建议用 `IDENTITY` 替代）
* 使用 `UUID v1`（包含 MAC 地址，泄露信息）
* 使用业务字段作为主键（订单号、手机号等）

### 4.7 乐观锁字段

对存在并发更新的核心实体（余额、订单状态、库存）必须启用乐观锁：

```sql
-- 方式一：版本号（推荐，语义清晰）
version  INT NOT NULL DEFAULT 0

UPDATE trade.orders
   SET status = 3, version = version + 1, updated_at = NOW()
 WHERE id = ? AND version = ?;
-- 根据 affected_rows 判断是否冲突

-- 方式二：ETag / 时间戳（可选，用于 HTTP 条件请求场景）
etag     VARCHAR(64) NOT NULL
```

* 禁止使用 `SELECT FOR UPDATE` 替代乐观锁做跨行业务编排（易引发死锁与长事务）
* 余额扣减、订单状态机流转必须使用乐观锁或幂等键双保险

### 4.8 审计 / 日志表标准字段

`log_*` 表除 §4.1 必备字段外，还必须包含：

| 字段  | 类型  | 说明  |
|-----|-----|-----|
| `trace_id` | `VARCHAR(64) NOT NULL` | 全链路 trace id（W3C traceparent 的 trace-id 部分） |
| `operator` | `VARCHAR(64)` | 操作人 uid 或服务名（系统操作用 `system:{service}`） |
| `operator_ip` | `INET` | 来源 IP（用户操作必填） |
| `action` | `VARCHAR(64) NOT NULL` | 动作类型（`create` / `update` / `delete` / 业务语义） |
| `target_type` | `VARCHAR(64) NOT NULL` | 被操作对象类型（如 `order` / `wallet`） |
| `target_id` | `VARCHAR(64) NOT NULL` | 被操作对象 ID |
| `before` | `JSONB` | 变更前快照（UPDATE / DELETE） |
| `after` | `JSONB` | 变更后快照（CREATE / UPDATE） |
| `extra` | `JSONB` | 扩展信息（备注、来源、请求参数等） |

* 表必须按月或按季度分区（见 §5.1）
* 禁止 UPDATE / DELETE，只允许 INSERT

### 4.9 CREATE TABLE 标准模板

新建表请复制以下模板并删除不适用项，确保必备字段、索引、COMMENT 一次到位：

```sql
-- migrations/000123_create_trade_orders.up.sql
SET search_path TO trade;

CREATE TABLE trade.orders (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    -- 多租户必备
    platform_id     VARCHAR(32)  NOT NULL,
    uid             VARCHAR(64)  NOT NULL,

    -- 业务字段
    symbol          VARCHAR(32)  NOT NULL,
    side            SMALLINT     NOT NULL,          -- 1=买 2=卖
    order_type      SMALLINT     NOT NULL,          -- 1=限价 2=市价
    price           NUMERIC(38,18) NOT NULL,
    quantity        NUMERIC(38,18) NOT NULL,
    filled_qty      NUMERIC(38,18) NOT NULL DEFAULT 0,
    status          SMALLINT     NOT NULL DEFAULT 1,

    -- 并发控制
    version         INT          NOT NULL DEFAULT 0,

    -- 必备审计字段
    created_at      TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ(6) NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ(6),
    created_by      VARCHAR(64),
    updated_by      VARCHAR(64),

    CONSTRAINT ck_orders_positive_qty  CHECK (quantity > 0),
    CONSTRAINT ck_orders_valid_side    CHECK (side IN (1,2))
);

-- 索引（platform_id 前缀强制）
CREATE INDEX CONCURRENTLY idx_orders_platform_uid_created
    ON trade.orders (platform_id, uid, created_at DESC);
CREATE INDEX CONCURRENTLY idx_orders_platform_symbol_status
    ON trade.orders (platform_id, symbol, status)
    WHERE deleted_at IS NULL;

-- COMMENT
COMMENT ON TABLE  trade.orders             IS '现货订单表';
COMMENT ON COLUMN trade.orders.status      IS '订单状态：1=待撮合 2=部分成交 3=完全成交 4=已取消 5=已过期';
COMMENT ON COLUMN trade.orders.side        IS '方向：1=买 2=卖';
COMMENT ON COLUMN trade.orders.version     IS '乐观锁版本号';

-- 权限
GRANT SELECT          ON trade.orders TO trade_service_read;
GRANT SELECT, INSERT, UPDATE ON trade.orders TO trade_service_write;
```


---

## 5. 分区与分片

### 5.1 分区表策略

满足以下任一条件的表**必须**设计分区：

| 触发条件 | 推荐分区方式 |
|------|--------|
| 预计行数 > 5000 万 / 数据量 > 100 GB | `RANGE` 分区（按时间） |
| 日志 / 审计 / 流水类（`log_*` / `hist_*`） | `RANGE` 分区（按月 / 季度） |
| 多租户强隔离场景 | `LIST` 分区（按 `platform_id`） |
| 数据均匀分布、查询按主键 | `HASH` 分区（8 / 16 / 32 片） |

**约束：**

* 分区键必须参与主键（PG 约束），通常组合为 `(id, created_at)` 或 `(id, platform_id)`
* 子分区命名：`{父表}_{分区特征}` → `orders_2024q1`, `log_user_login_p0`
* 分区创建、归档使用 `pg_partman` 或业务定时任务，禁止手工维护
* 历史分区超过保留期（默认 2 年）由归档作业迁移至 `hist_*` 或下线存储，Migration 中登记
* 查询必须带分区键，否则触发全分区扫描；DBR 审查时校验

### 5.2 分库分表（Sharding）

当单库容量 / TPS 超出单节点承载（参考阈值：容量 > 2 TB 或写入 > 20K TPS）时启用分片：

* 分片键选择优先级：`platform_id` > `uid` > 业务核心 ID（如 `order_id`）
* **禁止跨片 JOIN**；跨片聚合查询走数据仓库或 CQRS 只读视图
* 路由由中间件或应用层统一封装，业务代码禁止拼接分片库名
* 分片扩容需提前规划（一致性哈希 / 虚拟分片），避免全量数据迁移
* 分片库必须独立命名（`exchange_trade_shard_00` \~ `exchange_trade_shard_15`），Schema 结构保持一致
* 跨分片事务禁用，改用 Saga / 本地消息表实现最终一致


---

## 6. Migration 管理

### 6.1 工具

* Go 项目使用 `golang-migrate` 或 `goose`
* Migration 文件存放在 `migrations/` 目录

### 6.2 文件命名

```
migrations/
├── 000001_create_users_table.up.sql
├── 000001_create_users_table.down.sql
├── 000002_add_users_phone_column.up.sql
├── 000002_add_users_phone_column.down.sql
```

### 6.3 变更规则

* 每个变更一个独立 Migration 文件，禁止修改已发布的 Migration
* 必须同时提供 `up` 和 `down` 脚本
* 大表 DDL（> 100 万行）必须使用在线 DDL 工具（`pg_repack` / 业务灰度双写）
* 数据回填在独立 Migration 中执行，与 DDL 分开
* 禁止一次 Migration 同时修改多张无关表

### 6.4 审批流程

| 变更类型 | 审批要求 |
|------|------|
| 新增表  | Tech Lead 审批 |
| 新增字段/索引 | 同组开发者 Review |
| 修改字段类型 | Tech Lead + DBA 审批 |
| 删除表/字段 | Tech Lead + DBA + 产品确认 |
| 大表变更(>1M行) | DBA 专项评审 |

### 6.5 设计评审（DBR）Checklist

新表上线前必须通过以下检查项（由 Tech Lead + DBA 共同签字）：

**命名与结构**

- [ ] 表名 / 字段名 / 索引名符合 §3 命名规范
- [ ] 包含 §4.1 必备字段（`id` / `created_at` / `updated_at` / `deleted_at`）
- [ ] 多租户表包含 `platform_id` / `uid` 且索引前缀正确（§4.5）
- [ ] 所有表和字段均有 COMMENT（§3.9）

**字段与类型**

- [ ] 金额 / 数量字段类型符合 §4.4（区分法币与加密货币）
- [ ] 时间字段统一 `TIMESTAMPTZ(6)`
- [ ] 枚举字段使用 `SMALLINT` 并在 COMMENT 中枚举含义
- [ ] 字段数 ≤ 30，超出已规划垂直拆分

**索引与性能**

- [ ] 索引数量 ≤ 8，组合索引符合最左前缀
- [ ] 所有外部关联字段（`*_id`）均建索引
- [ ] 查询模式与索引匹配，已通过 `EXPLAIN ANALYZE` 验证
- [ ] 大表（> 5000 万行）已规划分区（§5.1）

**并发与安全**

- [ ] 高并发更新字段已配置乐观锁（§4.7）
- [ ] 敏感字段加密（§8.1）
- [ ] Role 权限按服务维度分配
- [ ] 无外键约束（§4.2）

**容量与运维**

- [ ] 已估算 1 年 / 3 年容量
- [ ] 已规划归档 / 清理策略（§9.4）
- [ ] 监控接入（慢查询、容量、TPS）

### 6.6 压测与容量估算

新表 / 重要变更上线前必须提供：

* **容量估算**：单行大小 × 预估行数（含增长率），结果记入 Migration PR
* **写入压测**：按预估 TPS 的 3 倍压测，确认不会引发锁等待或索引瓶颈
* **查询压测**：核心查询 P99 ≤ **100ms**，慢查询 ≤ **200ms**（§7.7）
* **回归测试**：关联表 JOIN / 写入链路在压测环境跑通
* 压测报告归档至 DBR 工单，不达标禁止合入


---

## 7. 查询规范

### 7.1 基础要求

* 禁止 `SELECT *`，必须指定字段
* 禁止在循环中执行查询（N+1 问题），使用批量查询或 JOIN
* 分页查询大数据量时使用游标分页（`WHERE id > ? ORDER BY id LIMIT ?`），禁止 OFFSET 大翻页
* 事务范围最小化，持锁时间尽量短
* 慢查询阈值：**200ms**，超出需优化并记录

### 7.2 批量操作

* 批量写入使用 `INSERT ... ON CONFLICT` 或 `COPY`，单批次 ≤ 1000 行
* 批量更新 / 删除必须分批，单批 ≤ 1000 行，批间 `sleep` 避免复制延迟飙升

### 7.3 游标分页标准写法

```sql
-- ✅ 推荐
SELECT id, symbol, created_at FROM trade.orders
 WHERE platform_id = $1 AND uid = $2 AND id < $last_id
 ORDER BY id DESC
 LIMIT 50;

-- ❌ 禁止（OFFSET 大翻页）
SELECT ... ORDER BY id LIMIT 50 OFFSET 100000;
```

### 7.4 事务与锁

* 默认隔离级别 `READ COMMITTED`；涉及读写一致性的业务（对账、扣减）使用 `REPEATABLE READ`，禁止 `SERIALIZABLE`（性能损失大）
* `SELECT FOR UPDATE` 仅限明确的资源锁（如秒杀扣减库存），必须在短事务内释放
* **加锁顺序**：跨表加锁必须按固定顺序（字典序表名或业务优先级），避免循环等待
* 死锁告警接入告警系统，频次 > 10 次 / 小时触发排障
* 长事务（> 10s）监控告警，自动 kill 超过 30s 的事务

### 7.5 慢查询监控

* 强制开启 `pg_stat_statements`，每日导出 Top 50 慢查询报表
* 慢查询阈值：**200ms**（P99）；超出阈值必须产出优化单
* 新慢查询首次出现 24 小时内分配 Owner
* 优化闭环：定位 → 执行计划分析 → 索引 / SQL 重写 → 压测验证 → 上线 → 效果复核
* 告警阈值：单 SQL TPS > 1000 且 P99 > 500ms，立即告警


---

## 8. 数据安全

### 8.1 敏感字段加密

* 敏感字段（手机号、身份证号、银行卡号、API Secret）存储时使用 `pgcrypto` 或应用层 AES-256-GCM 加密
* 加密字段命名加 `_enc` 后缀（如 `id_card_enc`），对应哈希用于匹配检索的字段加 `_hash` 后缀
* 密钥由 KMS 管理，禁止硬编码

### 8.2 权限管理

* 数据库账号按服务粒度分配，最小权限原则（读 / 写 / DDL 分离）
* 生产数据库禁止开发人员直连，通过堡垒机 + 审计日志访问
* DDL 操作只允许通过 CI Pipeline 执行，人工执行需双人复核

### 8.3 备份与恢复

* 定期备份：全量每日 + 增量每小时，备份保留 **30 天**
* 每月至少演练一次灾难恢复流程（RPO ≤ 1 小时，RTO ≤ 4 小时）
* 备份文件加密存储于异地对象存储

### 8.4 审计

* 开启 `pgaudit`，记录所有 DDL / 敏感表 DML
* 审计日志保留 ≥ 6 个月，写入 `audit` schema 并只追加
* 高敏操作（删库、删表、批量更新 / 删除）实时告警

### 8.5 数据脱敏

生产数据下发到测试 / 开发环境必须脱敏，规则：

| 字段类型 | 脱敏策略 |
|------|------|
| 手机号 / 邮箱 | 保留首尾，中间替换星号 / 按 hash 重写 |
| 身份证 / 银行卡 | 全量随机重写，保留校验位格式 |
| 姓名 / 地址 | 替换为同长度随机字符串 |
| 余额 / 金额 | 乘以随机扰动系数（0.5 \~ 1.5），保留精度格式 |
| `platform_id` / `uid` | 全局重映射，保持关联一致 |
| API Key / Secret | 直接清空或替换为测试固定值 |

* 脱敏通过 DBA 提供的 ETL 作业统一执行，禁止开发手工 dump
* 开发 / 测试环境禁止导入未脱敏的生产数据，违者记录考核


---

## 9. 运维规范

### 9.1 VACUUM / autovacuum

* 默认依赖 autovacuum；高写入表（> 1000 TPS）单独覆写参数：

  ```sql
  ALTER TABLE trade.orders SET (
      autovacuum_vacuum_scale_factor = 0.02,
      autovacuum_analyze_scale_factor = 0.01,
      fillfactor = 85
  );
  ```
* 禁止手工 `VACUUM FULL`（锁表），使用 `pg_repack` 或 `VACUUM (FREEZE, VERBOSE)`
* 表膨胀率 > 30% 触发告警，由 DBA 安排 repack

### 9.2 索引维护

* 索引膨胀率 > 30% 触发重建，使用 `REINDEX INDEX CONCURRENTLY`
* 每季度 Review 一次 `pg_stat_user_indexes`，下线零命中 / 低命中索引
* 新建索引一律使用 `CREATE INDEX CONCURRENTLY`
* 删除索引前先设置 `DISABLE`（软下线），观察一周无影响再 DROP

### 9.3 读写分离

* 写 / 强一致读必须走主库；弱一致读（报表、行情、历史查询）走只读副本
* 应用层在 DAO / Repository 层显式标注读写路由，禁止隐式依赖
* 副本复制延迟阈值：**1s**，超阈值告警并自动摘除副本流量
* 账单、资金对账类查询即使是读也走主库，避免复制延迟导致对账错误
* 副本不得承载长分析查询（> 30s），分析类走数仓 / OLAP

### 9.4 数据归档与冷热分离

* 在线表保留窗口：核心交易 2 年，日志 / 审计 6 个月，行情聚合永久
* 归档流程：

  
  1. 每月定时任务将过期分区切出，写入 `hist_*` 表或对象存储（Parquet）
  2. `hist_*` 表按月分区，读走独立只读副本
  3. 切出后 7 天确认无查询依赖，再从主库 DROP 分区
* 归档表必须保留与原表一致的 schema 结构，便于回查
* 对象存储归档启用生命周期策略：3 个月热 → 2 年温 → 7 年冷（Glacier）


---

## 10. Kafka 事件命名

> **中间件命名（Middleware）—— 由 IDP 控制。** 所有 Topic 必须经 IDP（内部开发者平台）申请、审批后下发；禁止应用直连 Broker 自助创建。

### 10.1 Topic 命名格式

**项目内 Topic：** `[环境]_[服务名]_[业务语义]_[动作]`

| 维度  | 规范  | 示例  |
|-----|-----|-----|
| 环境  | `prod` / `stage` / `dev` | `prod` |
| 服务名 | 发起方服务短名（不含 `-service` / `-job` 后缀） | `order`、`pay`、`wallet`、`matching` |
| 业务语义 | 事件主语 | `payStatus`、`deposit`、`trade` |
| 动作  | 过去式动词 | `updated`、`confirmed`、`matched` |

**示例：** `prod_order_payStatus_updated`、`prod_wallet_deposit_confirmed`、`prod_matching_trade_matched`

**跨项目 Topic：** `[环境]_[发起方服务]_to_[接收方服务]_[业务语义]`（模板 `env_A_2_B_biz`）

**示例：** `prod_pay_to_order_record`、`prod_order_to_wallet_settlement`、`prod_risk_to_dw_freeze`

### 10.2 通用约束

**禁止：**

* 使用 `.` 作为分隔符（如 `order.new.BTCUSDT`）
* 省略环境前缀
* 将 `platform_id` / `symbol` 编入 topic 名（应放入消息 payload / header）
* 应用绕过 IDP 自行创建 / 修改 / 删除 Topic

**消息必带头部：** `traceparent`（W3C Trace Context）、`platform_id`、`idempotency_key`。

### 10.3 消息 Schema 管理

* 所有 Topic 的消息体必须在 **Schema Registry**（Confluent / Apicurio）注册
* 格式统一为 **Protobuf**（首选）或 **Avro**，禁止裸 JSON
* Schema 演进遵循 `BACKWARD` 兼容策略：只允许新增可选字段，禁止删除 / 修改已有字段语义
* Schema 版本号与 Topic 解耦，在消息 header 中携带 `schema_id`
* Schema 定义随服务代码提交 Git，CI 校验与 Registry 同步

### 10.4 幂等消费与死信队列

* 消费端必须实现幂等：基于 `idempotency_key` 或 `{topic, partition, offset}` 去重
* 消费失败重试策略：最多 **3 次**，指数退避（1s / 5s / 30s）
* 超出重试次数进入死信队列（DLQ），命名：`{原 topic}_dlq` → `prod_wallet_deposit_confirmed_dlq`
* DLQ 必须接入告警，由业务 Owner 24 小时内处理
* DLQ 消息保留 ≥ 7 天


---

## 11. Redis Key 命名

### 11.1 Key 命名

**格式：** `{业务域}:{模块}:{唯一标识}[:{子键}]`

| 维度  | 规范  | 示例  |
|-----|-----|-----|
| `业务域` | CHEATSHEET §2.5.6 业务域短名 | `rewards`、`asset`、`risk` |
| `模块` | 域内二级功能 | `vip`、`balance`、`limit` |
| `唯一标识` | 业务主键（通常 `{platform_id}:{uid}` 或 `{platform_id}:{symbol}`） | `platform_A:10001` |
| `子键` | 可选细分 | `coin`、`counter` |

示例：`rewards:vip:platform_A:10001`、`asset:balance:platform_A:10001:BTC`、`risk:limit:platform_A:10001:withdraw_daily`

**约束：**

* 所有 key 必须带 TTL（无 TTL 需在设计文档中特别说明）
* key 不得包含空格、大写字母、中文
* 分片建议：以 `{业务域}:{模块}:` 前缀做 Hash Slot，便于同模块 mget / pipeline

### 11.2 大 Key / 热 Key 规约

| 限制  | 阈值  |
|-----|-----|
| 单 key value 大小 | ≤ **10 KB**（超出需拆分或改存储） |
| Hash / List / Set / ZSet 元素数 | ≤ **5000** |
| 单 key QPS | ≤ **5000**（超出视为热 Key，需本地缓存 + 多副本） |

* 大 Key 扫描每周执行一次，超限 key 由 Owner 7 天内整改
* 热 Key 识别接入监控（`redis-cli --hotkeys` 或 Proxy 层采样）
* 禁止在单 key 存储整张表 / 大集合，改用 Hash 分片（`{key}:{bucket}`）

### 11.3 分布式锁命名

**格式：** `lock:{业务域}:{资源类型}:{资源 ID}`

示例：`lock:wallet:withdrawal:10001`、`lock:trade:order:10001`

**约束：**

* TTL 必须设置（默认 10s，依业务调整）
* 使用 Redlock 或单实例 `SET NX PX` + Lua 释放脚本
* 锁持有方必须在 value 中写入 `{service}:{instance}:{request_id}`，释放前校验
* 禁止用作长事务锁（> 30s），应改异步任务 + 状态机

### 11.4 Pub/Sub Channel 命名

**格式：** `pubsub:{env}:{业务域}:{事件}`

示例：`pubsub:prod:market:symbol_update`、`pubsub:prod:risk:rule_reload`

* Pub/Sub 仅用于无持久化要求的即时广播（配置热加载、行情推送）
* 需持久化 / 重放场景必须使用 Kafka（§10），禁止用 Pub/Sub 替代
* 订阅端必须容忍消息丢失（Redis Pub/Sub 不保证送达）