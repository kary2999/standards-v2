---
title: "全局统一字段命名词典（MySQL / Kafka / Redis / REST JSON）"
version: "2.0.0-draft"
last_modified: "2026-04-30"
source: "技术规范.2026.04.29 / 全局统一字段命名规范.md"
---

# 全局统一字段命名规范

# 全平台数据库字段统一命名词典

> v2.0.0-draft | 适用：MySQL 8.4 / Kafka payload / Redis Key / REST JSON（Go + Gin 技术栈）
>
> 关联：[database.md](database.md)（MySQL 设计规范）、[api-doc.md](api-doc.md)、[kafka.md](kafka.md)、[redis.md](redis.md)
>
> **风格基线**：snake_case + 业务时间 `_time`（毫秒）+ TINYINT 状态 + `is_` 布尔。
>
> **v2.0.0 变更（2026-10-22）**：技术栈切换为 MySQL 8.4。（1）时间：`create_time` / `update_time`（毫秒 BIGINT）**必备**，`created_at` / `updated_at`（v2.2 起为 `DATETIME(6)`，此处原为 `TIMESTAMP`）由库维护，其余业务时间点统一 `{动作}_time` 毫秒 BIGINT；（2）主键 `id` 为自增 `BIGINT UNSIGNED`，另设业务 ID `oid`（`CHAR(36)` UUIDv7，v2.1 调整）；跨表引用字段用 `{entity}_oid`；（3）取消软删除（无 `deleted_at`）与 `created_by` / `updated_by`；（4）类型全部换为 MySQL 类型；（5）REST JSON 与数据库一致为 snake_case。
>
> **v1.0.1 变更（2026-04-30）**：与《数据库设计规范》v1.1.1 双向锚定 ——（1）`uid` 作为唯一用户 ID 命名（数据库规范同步修订）；（2）裸 `version` 让位给乐观锁、业务版本必须前缀；（3）`is_deleted` / `available` / `frozen` / `operator` / `operator_ip` 在规范示例语境下视为合法，词典软化对应硬禁；（4）缩写白名单收紧到 5 个核心；（5）新增附录 C 与规范的优先级。


---

## 0. 总则


1. **一个概念，一种命名**。同语义全平台只保留一个字段名。
2. **全平台 snake_case**。DB 列、Kafka payload、REST JSON 一律 snake_case，不做驼峰转换。
3. **后缀表达类型**（`_at` / `_amount` / `_status` / `_count`），前缀仅在同表冲突时使用。
4. **禁止过短缩写**。白名单：`id` / `url` / `ip` / `uid` / `ts`（与 [database.md](database.md) §3.1 一致）。`tx_hash` / `tx_amount` 中的 `tx` 视为"区块链交易"复合形态前缀，不作裸缩写使用。
5. **跨域引用必须带域前缀**（`wallet_request_id`、`c2c_order_id`），裸字段（`request_id`）禁止跨域。
6. **新表新字段必须遵守**；存量表违规进 §10 豁免清单，随大版本迁移。
7. **与 [database.md](database.md) 冲突时**：以 database.md 为准（时间字段、主键、软删除等已在其中重新定义），详见**附录 C**。


---

## 1. ID / 标识类

### 1.1 主键 / 关联

| 字段  | 类型  | 说明  |
|-----|-----|-----|
| `id` | `BIGINT UNSIGNED AUTO_INCREMENT` | 库内物理主键，每表必备，**仅本表内部使用**，不进 API / Kafka / Redis，不被其他表引用（见 [id-generation.md](id-generation.md)） |
| `oid` | `CHAR(36)` | 业务 ID，UUIDv7，小写 36 位，应用层生成，每表必备，带 `uk_{表}_oid`；API / Kafka / Redis / 日志里实体自身的 ID 就叫 `oid` |
| `{entity}_oid` | `CHAR(36)` | 逻辑外键：存对方表的 `oid`，使用**单数实体名**；禁用 DB-level FOREIGN KEY；**不要**叫 `{entity}_id`（会和对方自增 `id` 混淆） |

**示例：** `account_oid` / `order_oid` / `position_oid`（✅ 单数）；`accounts_oid` / `orders_oid`（❌ 复数）；`order_id` 存 `oid`（❌）。

**禁用：** `pk` / `seq_id` / `auto_id` / `row_id`、非 `id` 列使用 `AUTO_INCREMENT`、`INT` 自增（必须 `BIGINT UNSIGNED`）、表名直接拼 `_oid`（如 `accounts_oid` 应为 `account_oid`）。

> **特例：用户实体不走** `**{entity}_id**` **通式**，而是统一为 `uid`（见 §1.2）。即全平台**禁止**出现 `user_id` 字段名。

### 1.2 多租户三件套（ADR-001）

| 字段  | 类型  | 可空  |
|-----|-----|-----|
| `platform_id` | `VARCHAR(32)` | NOT NULL |
| `uid` | `VARCHAR(64)` | NOT NULL |
| `org_id` | `VARCHAR(64)` | NULL |

**禁用别名：**

* `user_id` / `u_id` / `userId` / `member_id` / `customer_id` → `**uid**`
* `tenant_id` / `brand_id` / `platform` / `platform_code`/ channel_id → `**platform_id**`
* `organization_id` / `institution_id` → `**org_id**`

#### 为什么选 `uid` 而不是 `user_id`

| #   | 理由  | 说明  |
|-----|-----|-----|
| 1   | **行业既定术语** | Binance / OKX / Bitget / Bybit 的官方文档、客服 SOP、用户互转界面统一称 **UID**；用户工单也是"发我你的 UID"。建表叫 `user_id` 会造成"内外两套词" |
| 2   | **多** `**_id**` **字段中识别主体更快** | 一张订单表常含 `order_id` / `account_id` / `position_id` / `client_order_id` / `parent_order_id`……如果归属人也叫 `user_id` 会被淹没。`platform_id × uid × org_id` 三件套是"主语级别"，`xxx_id` 是"客体级别"，视觉分层清晰 |
| 3   | **派生命名更干净（最关键）** | `operator_uid` / `referrer_uid` / `reviewer_uid` / `from_uid` / `to_uid` 两段式；用 `user_id` 会变成 `operator_user_id` / `referrer_user_id` 三段式，组合索引名 `idx_xxx_referrer_user_id_yyy` 容易突破 PG 63 字符限制 |
| 4   | **缩写白名单内** | `uid` 是 POSIX/Unix 历史定型缩写，与 `id` / `url` / `ip` / `pk` 同级，全行业无歧义，不属于"过短缩写"禁区 |
| 5   | **语义自带"全局唯一"** | `user_id` 在传统 Web2 / 阿里规范里多指"`users` 表内自增 BIGINT，scope 在单库"；`uid` = 跨 platform_id、跨业务线、生命周期不变的全局字符串身份（`VARCHAR(64)`，可承载 `USR_88001` / `did:ex:0xabc...` 等未来形态） |

**反方依据（保留参考）**：阿里 MySQL 手册推荐 `user_id`，国内 Web2 后端熟悉度高；Coinbase Exchange 历史也用 `user_id`。本规范因为面向**加密货币交易所**，行业术语对齐（#1）+ 派生命名简洁（#3）权重最高，故选 `uid`。

**最忌讳的不是选哪个，而是两个并存**——本规范一旦定下 `uid`，全平台禁止再出现 `user_id`。

**同表多 uid 场景：**

| 角色  | 字段  |
|-----|-----|
| 资源主体 | `uid` |
| 后台操作员 | `operator_uid` |
| 推荐人 | `referrer_uid` |
| 转入 / 收款 | `to_uid` |
| 转出 / 付款 | `from_uid` |
| 复核人 | `reviewer_uid` |

### 1.3 业务实体 ID

| 字段  | 类型  | 说明  |
|-----|-----|-----|
| `oid`（订单表自身） / `order_oid`（他表引用） | `CHAR(36)` | 平台侧订单 ID（即订单号） |
| `client_order_id` | `VARCHAR(64)` | 客户端订单 ID（幂等键） |
| `trade_fill_oid` | `CHAR(36)` | 成交流水 ID（现货 / 合约 / 期权统一） |
| `maker_order_oid` / `taker_order_oid` | `CHAR(36)` | 撮合双边引用 |
| `position_oid` | `CHAR(36)` | 仓位 ID |
| `account_oid` | `CHAR(36)` | 资产账户 ID |
| `liquidation_oid` | `CHAR(36)` | 强平事件 ID |
| `loan_oid` | `CHAR(36)` | 借贷主单 ID |
| `freeze_oid` | `CHAR(36)` | 资产冻结流水 ID |

**禁用别名：** `order_no` / `order_sn` / `order_id`（存 `oid` 时）→ 订单表自身用 `oid`，他表引用用 `order_oid`；`fill_id` / `match_id` / `execution_id` → `trade_fill_oid`；`acct_id` / `account_id` → `account_oid`。

### 1.4 资产 / 币种 / 交易对

| 字段  | 类型  | 取值示例 |
|-----|-----|------|
| `coin` | `VARCHAR(20)` | `BTC`、`USDT`、`SOL` |
| `currency` | `CHAR(3)` | `USD`、`EUR`（ISO 4217 法币） |
| `symbol` | `VARCHAR(40)` | `BTC-USDT` / `BTC-USDT-PERP` / `BTC-25APR26-50000-C` |
| `base_coin` / `quote_coin` | `VARCHAR(20)` | `symbol` 拆分冗余字段 |
| `chain` | `VARCHAR(32)` | `BTC` / `ETH` / `TRX` / `POLYGON` |
| `evm_chain_id` | `INT` | EVM ChainID 数值（1=ETH、56=BSC） |

**禁用别名：** `asset` / `token` / `crypto_currency`/item_id → `coin`；`fiat_currency` → `currency`；`pair` / `instrument_id` / `product_id` / `market`/ symbol_id → `symbol`；`network` → `chain`。

### 1.5 链上 / 钱包 / 资产操作

| 字段  | 类型  |
|-----|-----|
| `tx_hash` | `VARCHAR(128)` |
| `block_number` | `BIGINT` |
| `block_time` | `BIGINT UNSIGNED`（毫秒） |
| `from_address` / `to_address` | `VARCHAR(128)` |
| `wallet_request_id` | `VARCHAR(64)`（钱包域内部） |
| `asset_op_oid` | `CHAR(36)`（资产操作 ID 统一字段） |

**禁用别名：** `txid` / `transaction_hash` → `tx_hash`；`asset_release_tx_id` / `asset_settle_tx_id` / `asset_deduct_tx_id` → `**asset_op_oid**`。

### 1.6 幂等 / 链路追踪

| 字段  | 类型  | 用途  |
|-----|-----|-----|
| `idempotency_key` | `VARCHAR(128)` | 跨域幂等键 |
| `client_request_id` | `VARCHAR(64)` | 客户端 SDK 自带（1h 滚动唯一） |
| `wallet_request_id` | `VARCHAR(64)` | 钱包域专用，与上隔离 |
| `trace_id` | `VARCHAR(32)` | W3C trace-id |
| `span_id` | `VARCHAR(16)` | W3C span-id |

`**idempotency_key**` **格式：** `{biz_domain}:{action}:{primary_key}` 例：`asset:settle_trade_fill:88123456`、`rewards:rebate:88123456:USR_88001:1`。

**禁止裸** `**request_id**` **跨域使用**，必须加域前缀。

### 1.7 外部系统

| 字段  | 类型  |
|-----|-----|
| `external_id` | `VARCHAR(128)` |
| `provider` | `VARCHAR(32)` |
| `provider_ref_id` | `VARCHAR(128)` |


---

## 2. 时间 / 日期

### 2.1 通用规则

* 业务时间点统一 `BIGINT UNSIGNED`，**UTC 毫秒时间戳**，后缀 `_time`（`create_time` / `pay_time` / `expire_time`）
* 纯日期用 `DATE`，后缀 `_date`
* 仅 `created_at` / `updated_at` 使用 `DATETIME(6)`（由数据库维护，仅供运维参考，业务代码不读）；**全库禁用 `TIMESTAMP`**
* 禁止裸 `time` / `timestamp` / `ts` 作为业务列名
* `DATETIME` 不带时区，取值依赖会话时区：实例与连接必须统一 UTC，禁止 `SET time_zone`（见 [database.md](database.md) §4.1）；未来时间（到期、计划执行）仍用 `BIGINT` 毫秒

### 2.2 固定字段（每表必备）

| 字段 | 类型 | 谁写 | 说明 |
|-----|-----|-----|-----|
| `create_time` | `BIGINT UNSIGNED` | 应用层 | 创建时间，毫秒；业务逻辑、分页游标、分区键以它为准 |
| `update_time` | `BIGINT UNSIGNED` | 应用层 | 更新时间，毫秒；**每次 UPDATE 必写** |
| `created_at` | `DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)` | 数据库 | 创建时间（微秒），运维用 |
| `updated_at` | `DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)` | 数据库 | 更新时间（微秒），运维用；`DEFAULT` / `ON UPDATE` 精度必须与列一致 |

**不设** `deleted_at`（不使用软删除，见 [database.md](database.md) §4.4）。

**禁用别名：**

* `gmt_create` / `ctime` / `created` / `created_time` → `create_time`（毫秒）或 `created_at`（库维护）
* `gmt_modified` / `utime` / `modify_time` → `update_time` / `updated_at`
* `delete_time` / `gmt_deleted` / `is_deleted` / `deleted_at`（本项目不使用软删除）

### 2.3 业务流转时间（动作过去式 + `_time`）

`submitted_time` / `approved_time` / `rejected_time` / `executed_time` / `matched_time` / `filled_time` / `settled_time` / `paid_time` / `confirmed_time` / `processed_time` / `consumed_time` / `published_time` / `completed_time` / `canceled_time` / `failed_time` / `expired_time` / `repaid_time`

**拼写约束：** 美式 `canceled`（单 l），动作均为过去分词。

### 2.4 计划 / 截止时间

| 字段 | 含义 |
|-----|-----|
| `expire_time` | 计划过期时间（与 `expired_time` 已过期事实严格区分） |
| `valid_from` / `valid_to` | 有效期区间（毫秒） |
| `scheduled_time` | 计划执行时间 |
| `next_run_time` | 下次执行时间 |
| `due_date` | 到期日（`DATE`） |

**禁用：** `expires_at` / `expiry_at` / `valid_until` → `expire_time`；裸 `start_time` / `end_time` 表区间用 `valid_from` / `valid_to`，表事件用 `started_time` / `ended_time`。

### 2.5 Kafka / 日志中的时间

* 事件发生时间统一 `occurred_time`（毫秒），见 [kafka.md](kafka.md) §2.1
* 高频撮合内部可用纳秒 `match_ts_ns`（`BIGINT`），仅限撮合引擎内部，不对外暴露
* REST 响应外壳的 `timestamp` 为 ISO 8601 UTC 字符串（见 [api-doc.md](api-doc.md)）；业务时间字段仍为毫秒整数


---

## 3. 状态 / 类型

### 3.1 三分法

| 后缀  | 语义  | 类型  |
|-----|-----|-----|
| `status` | 生命周期流转（受状态机约束） | `TINYINT UNSIGNED` |
| `_state` | **仅状态机审计日志** `from_state` / `to_state` | `TINYINT UNSIGNED` |
| `_type` | 不变分类（创建后基本不改） | `TINYINT UNSIGNED` |
| `_mode` | 行为模式 / 算法选择 | `TINYINT UNSIGNED` |

**强制：** 业务表用 `status`，状态机日志用 `state`，禁止反过来；常量映射统一在 `域-lib` 包；禁止 VARCHAR / MySQL `ENUM` 存状态；常量值 `1` 起步，`0` 保留 UNKNOWN。

### 3.2 常用状态字段

`order_status` / `kyc_status` / `withdrawal_status` / `deposit_status` / `settlement_status` / `position_status` / `liquidation_status` / `loan_status` / `dispute_status` / `trade_status` / `card_status` / `auth_status` / `task_status`

裸 `status` 仅当表名已无歧义时允许（如 `c2c.orders.status`）。

### 3.3 类型字段

| 字段  | 取值示例 |
|-----|------|
| `order_type` | LIMIT / MARKET / STOP_LIMIT / LIMIT_MAKER |
| `account_type` | SPOT=1 / FUTURES=2 / OPTIONS=3 / EARN=4 / UCARD=5 / FEE_POOL=6 / UCARD_CREDIT=7 |
| `coin_type` | NATIVE / ERC20 / TRC20 / SPL / BEP20 |
| `chain_type` | EVM / UTXO / TRON / SOLANA |
| `fee_type` | TRADE / WITHDRAW / FUNDING / SETTLEMENT |
| `position_side` | LONG=1 / SHORT=2 / BOTH=3 |
| `order_side` | BUY=1 / SELL=2 |
| `time_in_force` | GTC / IOC / FOK / GTX |

**禁用：** `cate` / `category`（仅 cfg_\* 可保留）/ `kind` → `type`。


---

## 4. 金额 / 数量 / 价格

### 4.1 类型对照

| 类别  | 类型  | 后缀  |
|-----|-----|-----|
| 加密货币数量 / 余额 | `DECIMAL(38, 18)` | `_amount` / `_qty` / `_balance` |
| 加密货币价格 / 汇率 | `DECIMAL(38, 18)` | `_price` / `_rate` |
| 法币 / U 卡等价 | `DECIMAL(28, 8)` | `_amount` |
| 手续费率 / 利率 / APY | `DECIMAL(10, 8)` | `_rate` / `apy` / `apr` |
| 整数计数 | `BIGINT UNSIGNED` / `INT UNSIGNED` | `_count` |

### 4.2 金额（必带业务前缀）

`order_amount` / `trade_amount` / `fill_amount` / `notional_amount` / `fee_amount` / `rebate_amount` / `withdrawal_amount` / `deposit_amount` / `transfer_amount` / `freeze_amount` / `release_amount` / `principal_amount` / `interest_amount` / `repay_amount` / `liquidation_amount` / `premium_amount` / `payout_amount` / `cash_amount`

**禁止裸** `**amount**` 作列名。

### 4.3 数量

`quantity` / `original_qty` / `filled_qty` / `remaining_qty` / `liquidated_qty` / `available_qty`

**禁用：** `vol` / `size` / `amt` → `quantity` / `_qty`。

### 4.4 价格

`price` / `mark_price` / `index_price` / `last_price` / `settle_price` / `strike_price` / `liquidation_price` / `bankruptcy_price` / `avg_price` / `open_price` / `close_price` / `high_price` / `low_price`

### 4.5 余额

**推荐：** `available_balance` / `frozen_balance` / `total_balance` / `pending_balance` / `borrowed_balance` / `locked_balance`（带 `_balance` 后缀，可读性最佳）。

**兼容：** [database.md](database.md) §4.6 的金额类型约定列出的 `balance` / `available` / `frozen` 简短形式视为**合法等价**。同表内须保持单一风格，**不可混用**（如禁止一列 `available` 同时另一列 `frozen_balance`）。新表推荐带 `_balance` 后缀。

### 4.6 费率

`maker_rate` / `taker_rate` / `funding_rate` / `interest_rate` / `apy` / `apr` / `dcc_buffer_rate`

**禁用：** `fee_pct` / `rate_pct` / 裸 `rate`。


---

## 5. 布尔 / 标志

### 5.1 前缀规则

| 前缀  | 含义  | 示例  |
|-----|-----|-----|
| `is_` | 主语状态 | `is_active` / `is_test` / `is_maker_only` |
| `has_` | 拥有 / 存在 | `has_kyc` / `has_2fa` |
| `can_` | 能力（只读快照） | `can_trade` / `can_withdraw` |
| `allow_` / `_allowed` | 业务策略 | `is_market_order_allowed` |
| `enable_` / `_enabled` | 功能开关（运维侧） | `is_deposit_enabled` / `is_2fa_required` |

**强制：** 所有布尔列**必须** `**is_**` **开头**包住整个名字。✅ `is_deposit_enabled` / ❌ `deposit_enabled`；✅ `is_mfa_enabled` / ❌ `mfa_enabled`。

### 5.2 禁用

* `is_deleted` / `deleted_at` / `is_alive` —— 本项目**不使用软删除**（见 [database.md](database.md) §4.4），不得出现这类字段
* `is_disabled` — 用 `is_active = false`（保留正向语义）
* `xxx_flag` — 拆为具体业务布尔
* 三态属性禁用 `BOOLEAN + NULL` — 用 `_status` `TINYINT UNSIGNED`


---

## 6. 审计 / 版本

### 6.1 时间审计字段

本项目**不设** `created_by` / `updated_by`（也不设 `deleted_at`）。每表必备的时间字段见 §2.2：`create_time` / `update_time` / `created_at` / `updated_at`。

操作人追溯依赖：
* 审计日志表 `log_*` 的 `operator` 字段（[database.md](database.md) §4.9）
* 业务表需要记录“谁做的”时，用具体语义字段：`operator_uid` / `reviewer_uid` / `approved_by` 等（§6.2）
* 数据库 binlog

### 6.2 业务审计扩展

**业务表（非** `**log_\***`**）操作字段：** `operator_uid` / `reviewer_uid` / `approved_by` / `approved_time` / `rejected_by` / `rejected_time` / `op_source`（`web` / `app` / `api` / `internal-tool` / `system`）

**审计日志表（**`**log_\***` **前缀）字段：** 沿用 [database.md](database.md) §4.9 标准模板的 `operator` / `operator_ip` / `trace_id` / `action` / `target_type` / `target_id` / `before_snapshot` / `after_snapshot` / `extra`。审计日志表是规范定义的标准结构，**不再套用**业务表的 `operator_uid` / `client_ip` 命名。

### 6.3 版本字段（语义严格区分）

| 字段  | 类型  | 含义  |
|-----|-----|-----|
| `version` | `INT UNSIGNED` | **乐观锁**版本号（[database.md](database.md) §4.8 强制） |
| `schema_version` | `INT` | 行内 JSON / payload 的 schema 版本 |
| `rule_version` | `INT` | 业务规则快照版本（手续费 / 等级 / 风控） |
| `formula_version` | `INT` | 算法公式版本（云挖矿 / Earn 收益） |
| `config_version` | `INT` | 配置项版本 |

**强制：**

* **裸** `**version**` **永远表示乐观锁**，且仅用于这一语义；用法：`UPDATE ... SET version = version + 1 WHERE id = ? AND version = ?`
* **业务版本字段必须带语义前缀**（`rule_version` / `formula_version` / `config_version` / `schema_version`），禁止裸 `version` 作业务版本字段
* 这样区分使得"看到 `version` = 乐观锁"成为不可破坏的约定，无歧义


---

## 7. 备注 / 描述

| 字段  | 类型  | 用途  |
|-----|-----|-----|
| `name` | `VARCHAR(128)` | 实体名 |
| `display_name` | `VARCHAR(128)` | 展示名（多语言） |
| `description` | `TEXT` | 静态描述（配置 / 字典 / 活动介绍） |
| `remark` | `VARCHAR(500)` | 业务备注（用户填写、订单备注） |
| `review_note` | `TEXT` | 审核 / 复核备注 |
| `failure_reason` | `VARCHAR(500)` | 失败原因（机器写入） |
| `error_code` | `VARCHAR(64)` | `E_xxx_xxx` 格式 |
| `error_message` | `TEXT` | 错误详细信息 |

**禁用：** `comment`（与 SQL 关键字冲突）、裸 `note`（必须带前缀）、`memo`。


---

## 8. 网络 / 设备 / 安全

| 字段  | 类型  |
|-----|-----|
| `client_ip` | `VARBINARY(16)`（`INET6_ATON` / `INET6_NTOA`） |
| `server_ip` | `VARBINARY(16)` |
| `user_agent` | `VARCHAR(500)` |
| `device_id` | `VARCHAR(64)` |
| `device_type` | `VARCHAR(32)`（`ios` / `android` / `web` / `desktop`） |
| `app_version` | `VARCHAR(32)` |
| `country_code` | `CHAR(2)` (ISO 3166-1 alpha-2) |
| `lang` | `CHAR(5)` (BCP 47：`en` / `zh-CN` / `ja`) |
| `session_id` | `VARCHAR(64)` |
| `risk_score` | `INT`（0\~1000） |
| `geo_lat` / `geo_lng` | `DECIMAL(10,7)` |

**禁用别名：** `ip_addr` / `login_ip` / `source_ip` → `client_ip`；裸 `ua` → `user_agent`；`language` → `lang`。


---

## 9. 加密 / 区块链

| 字段  | 含义  |
|-----|-----|
| `encryption_key_id` | KMS / HSM 密钥 ID（90 天轮转） |
| `encryption_alg` | `AES-256-GCM` / `AES-256-CBC` |
| `iv` / `nonce` | 加密 IV / nonce |
| `signature` | 数字签名 |
| `pubkey` | 公钥  |
| `mpc_share_id` | MPC TSS 分片 ID |
| `confirmations` | 链上确认数 |
| `gas_used` / `gas_price` | EVM gas |
| `fee_paid_amount` / `fee_paid_coin` | 链上实际矿工费 + 币种 |
| `mempool_status` | `pending` / `confirmed` / `dropped` |


---

## 10. 现状冲突清单（迁移分级）

🔴 必须改 / 🟡 建议改 / 🟢 历史豁免

| 现存命名 | 出现位置 | 推荐  | 分级  |
|------|------|-----|-----|
| `user_id` / `u_id` / `userId` | 全平台扫描 | `**uid**` | 🔴（强制） |
| 裸 `request_id` | 11-earn / 03-dw | `wallet_request_id` / `earn_request_id` | 🔴  |
| `asset_release_tx_id` 等碎片 | 15-options / 17-c2c / 16-ops / 18-cloudmining | `asset_op_oid` | 🔴  |
| `mfa_enabled` | 02-puc | `is_mfa_enabled` | 🔴  |
| 裸 `version`（业务语义） | 16-ops / 18-cloudmining | `rule_version` / `formula_version` / `config_version` | 🔴  |
| `is_deleted` / `deleted_at` 软删除字段 | 防退化  | 不使用软删除，直接物理删除或用 `_status` 表达状态 | 🔴（ban） |
| `trade_id` | 15-options | `trade_fill_oid` | 🟡  |
| `note` / `comment` 散在 | 多域   | `remark` / `review_note` / `description` | 🟡  |
| 裸 `available` / `frozen` 不带 `_balance` | 06-asset 历史 | 推荐 `available_balance` / `frozen_balance`（同表风格一致即可） | 🟡（兼容） |
| 裸 `version`（乐观锁语义） | 多域   | **保留**（[database.md](database.md) §4.8 锚定） | 🟢  |
| `crypto_currency` / `fiat_currency` | 17-c2c | C2C 域内豁免 | 🟢  |
| `wallet_request_id` | 03-dw | 钱包域专用，保留 | 🟢  |
| `operator` / `operator_ip`（仅 `log_*` 表） | 各域审计表 | **保留**（[database.md](database.md) §4.9 标准模板） | 🟢  |


---

## 11. 前后缀速查

**后缀**

| 后缀  | 类型  | 语义  |
|-----|-----|-----|
| `_oid` | 同实体的 `oid` | 关联引用（`_id` 只用于 `platform_id` / `org_id` / `device_id` 等非实体主键标识） |
| `_time` | BIGINT UNSIGNED | 事件时刻（毫秒，UTC） |
| `created_at` / `updated_at` | DATETIME(6) | 库维护，仅运维参考 |
| `_date` | DATE | 纯日期 |
| `_amount` | DECIMAL(38,18) / DECIMAL(28,8) | 金额  |
| `_qty` | DECIMAL(38,18) | 数量  |
| `_balance` | DECIMAL(38,18) | 余额  |
| `_price` | DECIMAL(38,18) | 价格  |
| `_rate` | DECIMAL(10,8) | 比率  |
| `_count` | BIGINT UNSIGNED / INT UNSIGNED | 计数  |
| `_status` | TINYINT UNSIGNED | 流转状态 |
| `_state` | TINYINT UNSIGNED | 状态机日志专用 |
| `_type` | TINYINT UNSIGNED | 不变分类 |
| `_mode` | TINYINT UNSIGNED | 行为模式 |
| `_side` | TINYINT UNSIGNED | 方向  |
| `_reason` | VARCHAR / TINYINT UNSIGNED | 原因  |
| `_score` | INT | 评分  |

**前缀**

| 前缀  | 含义  |
|-----|-----|
| `is_` / `has_` / `can_` | 布尔（`is_` 首选） |
| `original_` | 原始值 |
| `total_` | 汇总值 |
| `min_` / `max_` | 边界  |
| `avg_` | 平均  |
| `from_` / `to_` | 双向引用 |
| `prev_` / `next_` | 前后  |
| `parent_` / `child_` | 树状  |


---

## 12. CI Lint 规则（草案）

```yaml
# error 级 —— 必须修复才能合入
- id: forbidden_synonym_uid
  level: error
  pattern: '\b(user_id|u_id|userId|member_id|customer_id)\s+(VARCHAR|BIGINT|CHAR)'
  fix: "用户主体字段全平台用 uid（§1.2）"

- id: forbidden_time_alias
  level: error
  pattern: '\b(gmt_create|gmt_modified|ctime|utime|created_time|modify_time|create_at)\b'
  fix: "用 create_time / update_time（毫秒）或 created_at / updated_at（库维护）"

- id: create_time_must_be_bigint
  level: error
  pattern: '\b(create_time|update_time)\s+(TIMESTAMP|DATETIME)'
  fix: "create_time / update_time 必须是 BIGINT UNSIGNED 毫秒时间戳"

- id: business_time_must_use_time_suffix
  level: error
  pattern: '\b(?!created_at\b|updated_at\b)\w+_at\s+(BIGINT|DATETIME|TIMESTAMP)'
  fix: "业务时间点用 {动作}_time BIGINT UNSIGNED（毫秒）；_at 仅限 created_at / updated_at 且为 DATETIME(6)"

- id: forbidden_timestamp_type
  level: error
  pattern: '(?<![A-Za-z_])TIMESTAMP(?![A-Za-z_])'
  fix: "全库禁用 TIMESTAMP（2038 上限、依赖会话时区）；created_at / updated_at 用 DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6)"

- id: forbidden_soft_delete
  level: error
  pattern: '\b(is_deleted|deleted_at|is_alive)\b'
  fix: "本项目不使用软删除（database.md §4.4）"

- id: forbidden_audit_by_columns
  level: error
  pattern: '\b(created_by|updated_by)\b'
  fix: "本项目不设 created_by / updated_by；操作人追溯用 log_* 审计表"

- id: forbidden_auto_increment
  level: error
  pattern: 'BIGSERIAL|GENERATED\s+ALWAYS\s+AS\s+IDENTITY|\b(TINY|SMALL|MEDIUM)?INT(\s+UNSIGNED)?\s+NOT\s+NULL\s+AUTO_INCREMENT'
  fix: "自增只允许 `id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT`；业务 ID 另用 oid CHAR(36) UUIDv7（id-generation.md）"

- id: bool_must_have_is_prefix
  level: error
  pattern: '\b\w+_(enabled|required|allowed)\s+(BOOLEAN|TINYINT)'
  fix: "布尔列必须 is_ 开头（如 is_mfa_enabled / is_2fa_required）"

- id: forbidden_mysql_enum_boolean
  level: error
  pattern: '\b(ENUM|BOOLEAN|BOOL|FLOAT|DOUBLE)\b\s*[(\s,]'
  fix: "枚举 / 布尔用 TINYINT UNSIGNED；金额用 DECIMAL"

- id: bare_amount_forbidden
  level: error
  pattern: '^\s*`?amount`?\s+(NUMERIC|DECIMAL)'
  fix: "禁止裸 amount，必须带业务前缀（trade_amount / fee_amount 等）"

- id: business_version_must_be_prefixed
  level: error
  description: "业务版本字段（非乐观锁）必须带语义前缀"
  custom_check: true
  fix: "业务版本用 rule_version / formula_version / config_version / schema_version；裸 version 保留给乐观锁"

- id: bare_time_forbidden
  level: error
  pattern: '\b(time|timestamp|ts)\s+(BIGINT|TIMESTAMP)'
  fix: "业务表禁止裸 time / timestamp / ts；用 _time 后缀"

# warning 级 —— 不阻塞合入但需 review
- id: balance_field_style_consistency
  level: warning
  description: "推荐 _balance 后缀；同表内必须风格一致"
  fix: "新表使用 available_balance / frozen_balance；旧表保持单一风格"

- id: audit_table_uses_standard_fields
  level: info
  description: "log_* 表沿用 database.md §4.9 的 operator / operator_ip / before_snapshot / after_snapshot 模板"
```

**PR 模板：**

- [ ] 字段已对照 §1\~§9
- [ ] 无禁用别名
- [ ] 布尔统一 `is_` / `has_` / `can_` 前缀
- [ ] 含 `create_time` / `update_time`（`BIGINT UNSIGNED` 毫秒）与 `created_at` / `updated_at`（`DATETIME(6)`）
- [ ] 业务时间点用 `{动作}_time` 毫秒 BIGINT
- [ ] 状态字段 `TINYINT UNSIGNED` + 域 lib 常量
- [ ] 无 `deleted_at` / `created_by` / `updated_by`；`id` 为 `BIGINT UNSIGNED AUTO_INCREMENT`，另有 `oid CHAR(36)` + `uk_*_oid`；引用字段为 `{entity}_oid`
- [ ] 表 / 字段已写 `COMMENT`
- [ ] 跨域字段使用域前缀


---

## 13. 豁免登记表

| 表 / 范围 | 字段  | 偏离规则 | 理由  | 有效期 | Owner |
|--------|-----|------|-----|-----|-------|
| **全平台业务表** | 裸 `version`（乐观锁） | 业务版本须带前缀 | [database.md](database.md) §4.8 锚定为 `version` | 永久  | 架构组   |
| **全平台审计表（**`**log_\***`**）** | `operator` / `operator_ip` | §6.2 / §8 | [database.md](database.md) §4.9 标准模板字段，业务表 `operator_uid` / `client_ip` 不适用于审计表 | 永久  | 架构组   |
| **加密货币余额表** | 裸 `available` / `frozen` / `balance` | §4.5 推荐带 `_balance` | [database.md](database.md) §4.6 列为合法等价；同表内须风格一致 | 永久  | Asset TL |


## 附录 A：与其他规范的边界

| 主题  | 由谁定义 |
|-----|------|
| 库 / 表 / 索引命名、DDL 风格 | [database.md](database.md) §3 |
| 字段类型选择（DECIMAL 精度 / 时间字段） | [database.md](database.md) §4.1 / §4.6 |
| 多租户字段（`platform_id` / `uid` / `org_id`） | [database.md](database.md) §4.7 |
| 必备字段（`id` / `create_time` / `update_time` / `created_at` / `updated_at`） | [database.md](database.md) §4.1 |
| 主键与分布式 ID | [id-generation.md](id-generation.md) + [database.md](database.md) §4.3 |
| 乐观锁字段命名（裸 `version`） | [database.md](database.md) §4.8 |
| 审计日志表标准字段 | [database.md](database.md) §4.9 |
| Kafka topic 命名 / 消息信封 | [kafka.md](kafka.md) |
| Redis Key 命名 | [redis.md](redis.md) |
| REST JSON 字段与类型 | [api-doc.md](api-doc.md) |
| **字段语义命名（uid / order_id / trade_fill_id / 业务流转** `**_time**` **等）** | **本词典** |
| **派生用户字段（**`**operator_uid**` **/** `**referrer_uid**` **/** `**from_uid**` **/** `**to_uid**` **等）** | **本词典** |
| **业务版本字段语义前缀（**`**rule_version**` **/** `**formula_version**` **/** `**config_version**`**）** | **本词典** |
| **备注字段四分（**`**description**` **/** `**remark**` **/** `**review_note**` **/** `**failure_reason**`**）** | **本词典** |

## 附录 B：词典维护流程


1. 新增 / 变更字段命名 → 在本词典开 PR
2. 架构组 review，必要时同步 `database.md`
3. 合入后版本号 +0.0.1（小修）/ +0.1.0（新增类别）/ +1.0.0（破坏性收敛）

## 附录 C：与 database.md 的优先级

当本词典与 [database.md](database.md) 出现差异时，按以下优先级裁决：

| 层级  | 内容  | 优先级 | 备注  |
|-----|-----|-----|-----|
| 1   | database.md §3 / §4 等**强制条款** | 最高  | 含必备字段、时间字段、主键、多租户、乐观锁、审计表 |
| 2   | 本词典 §1.2 用户 ID（`uid`）+ §6.3 业务版本前缀 + §6.2 业务表操作字段（`operator_uid` / `client_ip`） | 高   | |
| 3   | 本词典 §1\~§9 其他字段语义规则 | 中   | 在 database.md 未明确处填补 |
| 4   | 本词典 §10 现状冲突清单 / §13 豁免 | 低   | 域级最终落地依据 |

**实践快速记忆：**

* 看到 `version` → 乐观锁；看到 `xxx_version` → 业务版本
* `log_*` 审计表用 `operator` / `operator_ip`；业务表用 `operator_uid` / `client_ip`
* 没有软删除：永远不要 `deleted_at` / `is_deleted`
* 时间：`create_time` / `update_time` 毫秒必备；其他业务时间 `{动作}_time` 毫秒；`created_at` / `updated_at` 仅库维护
* 用户主体字段一律 `uid`，永远不要 `user_id` 字段
* 两个 ID：`id` 自增主键只在库内用；`oid`（UUIDv7，`CHAR(36)`）才是业务 ID；引用别的实体写 `{entity}_oid`
* 余额字段推荐 `_balance` 后缀；旧表沿用 `available` / `frozen` 不强迁，但同表内须风格一致

---

## 版本记录

| 版本  | 日期  | 摘要  |
|-----|-----|-----|
| v1.0.0 | 2026-04-30 | 初版；9 大类字段 + 17 项冲突 + 3 项豁免 |
| v1.0.1 | 2026-04-30 | 与《数据库设计规范》v1.1.1 双向锚定：(1) `uid` 作为唯一用户 ID（规范同步反向修订）；(2) 裸 `version` 让位乐观锁（规范 §4.7 既定），业务版本必须前缀；(3) `is_deleted` 软化为"仅作 `is_` 前缀示例"；(4) `available` / `frozen` 视为 `available_balance` 等价；(5) 审计表 `operator` / `operator_ip` 沿用规范 §4.8；(6) 缩写白名单收紧到 5 个；(7) 新增附录 C 优先级裁决 + 5 条快速记忆 |
| v2.0.0-draft | 2026-10-22 | 技术栈切换为 MySQL 8.4：时间字段改 `create_time`/`update_time`（毫秒）+ `created_at`/`updated_at`（TIMESTAMP）；业务时间 `_at`→`_time`；实体 ID 改 UUIDv7 `BINARY(16)`；取消软删除与 `created_by`/`updated_by`；类型换 MySQL；REST JSON 统一 snake_case |
| v2.2.0-draft | 2026-10-24 | `created_at` / `updated_at` 由 `TIMESTAMP` 改为 `DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6)`（`updated_at` 加 `ON UPDATE CURRENT_TIMESTAMP(6)`）；全库禁用 `TIMESTAMP`；lint 规则同步 |
| v2.1.0-draft | 2026-10-22 | ID 方案调整：`id` 保留自增 `BIGINT UNSIGNED`，新增 `oid CHAR(36)`（UUIDv7）作业务 ID；引用字段 `{entity}_id` → `{entity}_oid`；实体 ID 字典同步改名 |
