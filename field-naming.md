---
title: "全局统一字段命名词典（PG / Kafka / Redis / Proto / OpenAPI）"
version: "1.0.1"
last_modified: "2026-04-30"
source: "技术规范.2026.04.29 / 全局统一字段命名规范.md"
---

# 全局统一字段命名规范

# 全平台数据库字段统一命名词典

> v1.0.1 | 适用：PostgreSQL / Kafka payload / Redis Key / Proto / OpenAPI
>
> 关联：`数据库设计规范.md` v1.1.1+、`状态机设计规范.md`、`MASTER-DESIGN.md` ADR-001\~010
>
> **风格基线**：snake_case + `_at` 后缀 + SMALLINT 状态 + `is_` 布尔（与 Coinbase / 字节内部规范一致）。
>
> **v1.0.1 变更（2026-04-30）**：与《数据库设计规范》v1.1.1 双向锚定 ——（1）`uid` 作为唯一用户 ID 命名（数据库规范同步修订）；（2）裸 `version` 让位给乐观锁、业务版本必须前缀；（3）`is_deleted` / `available` / `frozen` / `operator` / `operator_ip` 在规范示例语境下视为合法，词典软化对应硬禁；（4）缩写白名单收紧到 5 个核心；（5）新增附录 C 与规范的优先级。


---

## 0. 总则


1. **一个概念，一种命名**。同语义全平台只保留一个字段名。
2. **存储 snake_case，对外 camelCase**。DB / Kafka / Proto field 全 snake_case，REST JSON 由网关转换。
3. **后缀表达类型**（`_at` / `_amount` / `_status` / `_count`），前缀仅在同表冲突时使用。
4. **禁止过短缩写**。白名单：`id` / `url` / `ip` / `uid` / `ts`（与《数据库设计规范》§2.5 一致）。`tx_hash` / `tx_amount` 中的 `tx` 视为"区块链交易"复合形态前缀，不作裸缩写使用。
5. **跨域引用必须带域前缀**（`wallet_request_id`、`c2c_order_id`），裸字段（`request_id`）禁止跨域。
6. **新表新字段必须遵守**；存量表违规进 §10 豁免清单，随大版本迁移。
7. **与《数据库设计规范》冲突时**：用户 ID 字段以本词典为准（已反向锚定规范 v1.1.1）；其余命名以《数据库设计规范》为准，详见**附录 C**。


---

## 1. ID / 标识类

### 1.1 主键 / 关联

| 字段  | 类型  | 说明  |
|-----|-----|-----|
| `id` | `BIGINT IDENTITY` | 物理主键，每表必备 |
| `{entity}_id` | 同实体主键类型 | 逻辑外键，使用**单数实体名**；禁用 DB-level FOREIGN KEY |

**示例：** `account_id` / `order_id` / `position_id`（✅ 单数）；`accounts_id` / `orders_id`（❌ 复数）。

**禁用：** `pk` / `seq_id` / `auto_id` / `row_id`、表名直接拼 `_id`（如 `accounts_id` 应为 `account_id`）。

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
| `order_id` | `BIGINT` | 平台侧订单 ID |
| `client_order_id` | `VARCHAR(64)` | 客户端订单 ID（幂等键） |
| `trade_fill_id` | `BIGINT` | 成交流水 ID（现货 / 合约 / 期权统一） |
| `maker_order_id` / `taker_order_id` | `BIGINT` | 撮合双边引用 |
| `position_id` | `BIGINT` | 仓位 ID |
| `account_id` | `BIGINT` | 资产账户 ID |
| `liquidation_id` | `BIGINT` | 强平事件 ID |
| `loan_id` | `BIGINT` | 借贷主单 ID |
| `freeze_id` | `BIGINT` | 资产冻结流水 ID |

**禁用别名：** `oid` / `order_no` / `order_sn` → `order_id`；`fill_id` / `match_id` / `execution_id` → `trade_fill_id`；`acct_id` → `account_id`。

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
| `block_time` | `TIMESTAMPTZ(6)` |
| `from_address` / `to_address` | `VARCHAR(128)` |
| `wallet_request_id` | `VARCHAR(64)`（钱包域内部） |
| `asset_op_id` | `BIGINT`（资产操作 ID 统一字段） |

**禁用别名：** `txid` / `transaction_hash` → `tx_hash`；`asset_release_tx_id` / `asset_settle_tx_id` / `asset_deduct_tx_id` → `**asset_op_id**`。

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

* 类型一律 `**TIMESTAMPTZ(6)**`，UTC 存储
* 后缀 `_at` 时间点；后缀 `_date` 纯日期 `DATE`
* 禁止裸 `time` / `timestamp` / `ts` 作为业务列名

### 2.2 三大固定字段

| 字段  | 缺省  |
|-----|-----|
| `created_at` | `NOW()` |
| `updated_at` | `NOW()`（trigger / ORM 维护） |
| `deleted_at` | NULL（NULL 即未删除） |

**禁用别名（任何场景都不允许）：**

* `create_time` / `gmt_create` / `ctime` / `created` / `created_time` → `**created_at**`
* `update_time` / `gmt_modified` / `utime` / `modify_time` → `**updated_at**`
* `delete_time` / `gmt_deleted` / `is_deleted` → `**deleted_at**`

### 2.3 业务流转时间（动作过去式 + `_at`）

`submitted_at` / `approved_at` / `rejected_at` / `executed_at` / `matched_at` / `filled_at` / `settled_at` / `paid_at` / `confirmed_at` / `processed_at` / `consumed_at` / `published_at` / `completed_at` / `canceled_at` / `failed_at` / `expired_at` / `repaid_at`

**拼写约束：** 美式 `canceled`（单 l），动作均为过去分词。

### 2.4 计划 / 截止时间

| 字段  | 含义  |
|-----|-----|
| `expire_at` | 计划过期时间（与 `expired_at` 已过期事实严格区分） |
| `valid_from` / `valid_to` | 有效期区间 |
| `scheduled_at` | 计划执行时间 |
| `next_run_at` | 下次执行时间 |
| `due_date` | 到期日（DATE） |

**禁用：** `expires_at` / `expiry_at` / `valid_until` → `expire_at`；裸 `start_time` / `end_time` 表区间用 `valid_from` / `valid_to`，表事件用 `started_at` / `ended_at`。

### 2.5 epoch 数值时间戳

仅限 Kafka payload / 高频日志，DB 业务表禁用。

| 字段  | 类型  |
|-----|-----|
| `event_ts_ms` | `BIGINT` |
| `ingest_ts_ms` | `BIGINT` |
| `match_ts_ns` | `BIGINT`（撮合内部） |


---

## 3. 状态 / 类型

### 3.1 三分法

| 后缀  | 语义  | 类型  |
|-----|-----|-----|
| `status` | 生命周期流转（受状态机约束） | `SMALLINT` |
| `_state` | **仅状态机审计日志** `from_state` / `to_state` | `SMALLINT` |
| `_type` | 不变分类（创建后基本不改） | `SMALLINT` |
| `_mode` | 行为模式 / 算法选择 | `SMALLINT` |

**强制：** 业务表用 `status`，状态机日志用 `state`，禁止反过来；常量映射统一在 `域-lib` 包；禁止 VARCHAR ENUM 字符串存状态；常量值 `1` 起步，`0` 保留 UNKNOWN。

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
| 加密货币数量 / 余额 | `NUMERIC(38, 18)` | `_amount` / `_qty` / `_balance` |
| 加密货币价格 / 汇率 | `NUMERIC(38, 18)` | `_price` / `_rate` |
| 法币 / U 卡等价 | `DECIMAL(28, 8)` | `_amount` |
| 手续费率 / 利率 / APY | `NUMERIC(10, 8)` | `_rate` / `apy` / `apr` |
| 整数计数 | `BIGINT` / `INT` | `_count` |

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

**兼容：** 《数据库设计规范》§3.4 加密货币例外条款列出的 `balance` / `available` / `frozen` 简短形式视为**合法等价**。同表内须保持单一风格，**不可混用**（如禁止一列 `available` 同时另一列 `frozen_balance`）。新表推荐带 `_balance` 后缀。

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

* `is_deleted` BOOLEAN 作为**软删除字段** — 软删除统一用 `deleted_at TIMESTAMPTZ IS NULL` 判断（《数据库设计规范》§3.1 强制必备字段）。规范 §2.5 表格中将 `is_deleted` 列作 `is_` 前缀的命名形态示例，**不得**作为软删除实际字段使用
* `is_disabled` — 用 `is_active = false`（保留正向语义）
* `xxx_flag` — 拆为具体业务布尔
* 三态属性禁用 `BOOLEAN + NULL` — 用 `_status` SMALLINT


---

## 6. 审计 / 版本

### 6.1 审计四件套

| 字段  | 类型  |
|-----|-----|
| `created_by` | `VARCHAR(64)` |
| `updated_by` | `VARCHAR(64)` |
| `created_at` | `TIMESTAMPTZ(6)` |
| `updated_at` | `TIMESTAMPTZ(6)` |

`**created_by**` **取值：**

* 用户操作 → `uid`
* 后台员工 → `op:{operator_uid}`
* 系统自动 → `system:{module}`（如 `system:matching-engine`）
* 跨服务 → `svc:{service-name}`

**禁用：** `creator` / `create_user` → `created_by`；`updater` → `updated_by`。

### 6.2 业务审计扩展

**业务表（非** `**log_\***`**）操作字段：** `operator_uid` / `reviewer_uid` / `approved_by` / `approved_at` / `rejected_by` / `rejected_at` / `op_source`（`web` / `app` / `api` / `internal-tool` / `system`）

**审计日志表（**`**log_\***` **前缀）字段：** 沿用《数据库设计规范》§4.8 标准模板的 `operator` / `operator_ip` / `trace_id` / `action` / `target_type` / `target_id` / `before` / `after` / `extra`。审计日志表是规范定义的标准结构，**不再套用**业务表的 `operator_uid` / `client_ip` 命名。

### 6.3 版本字段（语义严格区分）

| 字段  | 类型  | 含义  |
|-----|-----|-----|
| `version` | `INT` / `BIGINT` | **乐观锁**版本号（《数据库设计规范》§4.7 强制） |
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
| `client_ip` | `INET` |
| `server_ip` | `INET` |
| `user_agent` | `VARCHAR(500)` |
| `device_id` | `VARCHAR(64)` |
| `device_type` | `VARCHAR(32)`（`ios` / `android` / `web` / `desktop`） |
| `app_version` | `VARCHAR(32)` |
| `country_code` | `CHAR(2)` (ISO 3166-1 alpha-2) |
| `lang` | `CHAR(5)` (BCP 47：`en` / `zh-CN` / `ja`) |
| `session_id` | `VARCHAR(64)` |
| `risk_score` | `INT`（0\~1000） |
| `geo_lat` / `geo_lng` | `NUMERIC(10,7)` |

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
| `asset_release_tx_id` 等碎片 | 15-options / 17-c2c / 16-ops / 18-cloudmining | `asset_op_id` | 🔴  |
| `mfa_enabled` | 02-puc | `is_mfa_enabled` | 🔴  |
| 裸 `version`（业务语义） | 16-ops / 18-cloudmining | `rule_version` / `formula_version` / `config_version` | 🔴  |
| `is_deleted` BOOLEAN 作软删除字段 | 防退化  | `deleted_at TIMESTAMPTZ` | 🔴（ban） |
| `trade_id` | 15-options | `trade_fill_id` | 🟡  |
| `note` / `comment` 散在 | 多域   | `remark` / `review_note` / `description` | 🟡  |
| 裸 `available` / `frozen` 不带 `_balance` | 06-asset 历史 | 推荐 `available_balance` / `frozen_balance`（同表风格一致即可） | 🟡（兼容） |
| 裸 `version`（乐观锁语义） | 多域   | **保留**（《数据库设计规范》§4.7 锚定） | 🟢  |
| `crypto_currency` / `fiat_currency` | 17-c2c | C2C 域内豁免 | 🟢  |
| `wallet_request_id` | 03-dw | 钱包域专用，保留 | 🟢  |
| `operator` / `operator_ip`（仅 `log_*` 表） | 各域审计表 | **保留**（《数据库设计规范》§4.8 标准模板） | 🟢  |


---

## 11. 前后缀速查

**后缀**

| 后缀  | 类型  | 语义  |
|-----|-----|-----|
| `_id` | 同实体主键 | 关联引用 |
| `_at` | TIMESTAMPTZ(6) | 事件时刻 |
| `_date` | DATE | 纯日期 |
| `_amount` | NUMERIC(38,18) / DECIMAL(28,8) | 金额  |
| `_qty` | NUMERIC(38,18) | 数量  |
| `_balance` | NUMERIC(38,18) | 余额  |
| `_price` | NUMERIC(38,18) | 价格  |
| `_rate` | NUMERIC(10,8) | 比率  |
| `_count` | BIGINT / INT | 计数  |
| `_status` | SMALLINT | 流转状态 |
| `_state` | SMALLINT | 状态机日志专用 |
| `_type` | SMALLINT | 不变分类 |
| `_mode` | SMALLINT | 行为模式 |
| `_side` | SMALLINT | 方向  |
| `_reason` | VARCHAR / SMALLINT | 原因  |
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
  fix: "用户主体字段全平台用 uid（《字段命名词典》§1.2 + 《数据库设计规范》v1.1.1）"

- id: forbidden_synonym_created_at
  level: error
  pattern: '\b(create_time|gmt_create|ctime|created_time|create_at)\s+TIMESTAMP'
  fix: "用 created_at"

- id: forbidden_is_deleted_as_soft_delete
  level: error
  pattern: 'is_deleted\s+(BOOLEAN|BOOL)\s+NOT\s+NULL'
  fix: "软删除字段禁用 is_deleted BOOLEAN；用 deleted_at TIMESTAMPTZ（《数据库设计规范》§3.1）"

- id: bool_must_have_is_prefix
  level: error
  pattern: '\b\w+_(enabled|required|allowed)\s+BOOLEAN'
  fix: "布尔列必须 is_ 开头（如 is_mfa_enabled / is_2fa_required）"

- id: bare_amount_forbidden
  level: error
  pattern: '^\s*amount\s+(NUMERIC|DECIMAL)'
  fix: "禁止裸 amount，必须带业务前缀（trade_amount / fee_amount 等）"

- id: business_version_must_be_prefixed
  level: error
  description: "业务版本字段（非乐观锁）必须带语义前缀"
  custom_check: true
  fix: "业务版本用 rule_version / formula_version / config_version / schema_version；裸 version 保留给乐观锁"

- id: bare_time_forbidden
  level: error
  pattern: '\b(time|timestamp|ts)\s+(BIGINT|TIMESTAMPTZ)'
  fix: "业务表禁止裸 time / timestamp / ts；用 _at 后缀"

# warning 级 —— 不阻塞合入但需 review
- id: balance_field_style_consistency
  level: warning
  description: "推荐 _balance 后缀；同表内必须风格一致"
  fix: "新表使用 available_balance / frozen_balance；旧表保持单一风格"

- id: audit_table_uses_standard_fields
  level: info
  description: "log_* 表沿用规范 §4.8 的 operator / operator_ip / before / after 模板"
```

**PR 模板：**

- [ ] 字段已对照 §1\~§9
- [ ] 无禁用别名
- [ ] 布尔统一 `is_` / `has_` / `can_` 前缀
- [ ] 时间统一 `_at` 后缀 + `TIMESTAMPTZ(6)`
- [ ] 状态字段 `SMALLINT` + 域 lib 常量
- [ ] 表 / 字段已写 `COMMENT`
- [ ] 跨域字段使用域前缀


---

## 13. 豁免登记表

| 表 / 范围 | 字段  | 偏离规则 | 理由  | 有效期 | Owner |
|--------|-----|------|-----|-----|-------|
| `c2c.orders` | `crypto_currency` / `fiat_currency` | §1.4 | base/quote 同表消歧 | 永久  | C2C TL |
| `dw.wallet_log` | `wallet_request_id` | §1.6 | 钱包 SDK 协议解耦 | 永久  | DW TL |
| `options.trades` | `trade_id` | §1.3 | 历史命名，迁移成本高 | v2 重构前 | Options TL |
| **全平台业务表** | 裸 `version`（乐观锁） | 词典早期版 §6.3 提案 `row_version` | 《数据库设计规范》§4.7 锚定为 `version`；词典 v1.0.1 已收回 `row_version` 提案 | 永久  | 架构组   |
| **全平台审计表（**`**log_\***`**）** | `operator` / `operator_ip` / `before` / `after` | §6.2 / §8 | 《数据库设计规范》§4.8 标准模板字段，业务表 `operator_uid` / `client_ip` 不适用于审计表 | 永久  | 架构组   |
| **加密货币余额表** | 裸 `available` / `frozen` / `balance` | §4.5 推荐带 `_balance` | 《数据库设计规范》§3.4 加密货币例外条款列为合法等价示例；同表内须风格一致 | 永久  | Asset TL |


---

## 附录 A：与其他规范的边界

| 主题  | 由谁定义 |
|-----|------|
| 表 / Schema / 索引命名、DDL 风格 | 《数据库设计规范》§2 |
| 字段类型选择（NUMERIC 精度 / TIMESTAMPTZ） | 《数据库设计规范》§3.4 |
| 多租户字段（`platform_id` / `uid` / `org_id`） | 《数据库设计规范》§3.5 + ADR-001 |
| 必备字段（`id` / `created_at` / `updated_at` / `deleted_at` / `created_by` / `updated_by`） | 《数据库设计规范》§4.1 |
| 乐观锁字段命名（裸 `version`） | 《数据库设计规范》§4.7 |
| 审计日志表标准字段（`operator` / `operator_ip` / `before` / `after` 等） | 《数据库设计规范》§4.8 |
| Kafka topic 命名 | 《数据库设计规范》§7 |
| Redis Key 命名 | 《数据库设计规范》§8 |
| **字段语义命名（uid / order_id / trade_fill_id / 业务流转** `**_at**` **等）** | **本词典** |
| **派生用户字段（**`**operator_uid**` **/** `**referrer_uid**` **/** `**from_uid**` **/** `**to_uid**` **等）** | **本词典** |
| **业务版本字段语义前缀（**`**rule_version**` **/** `**formula_version**` **/** `**config_version**`**）** | **本词典** |
| **备注字段四分（**`**description**` **/** `**remark**` **/** `**review_note**` **/** `**failure_reason**`**）** | **本词典** |
| 状态机 `from_state` / `to_state` | 《状态机设计规范》§3 |

## 附录 B：词典维护流程


1. 新增 / 变更字段命名 → 在本词典开 PR
2. 架构组 review，必要时同步 `数据库设计规范.md` / `DESIGN-CHEATSHEET.md`
3. 合入后版本号 +0.0.1（小修）/ +0.1.0（新增类别）/ +1.0.0（破坏性收敛）
4. 同步在 `重构/ALIGNMENT-REPORT.md` 命名规范矩阵中登记

## 附录 C：与《数据库设计规范》的优先级（v1.0.1 新增）

当本词典与《数据库设计规范》v1.1.1 出现差异时，按以下优先级裁决：

| 层级  | 内容  | 优先级 | 备注  |
|-----|-----|-----|-----|
| 1   | 《数据库设计规范》§3 / §4 / §7 / §8 等**强制条款** | 最高  | 含必备字段、多租户、乐观锁、审计表、Kafka/Redis 规范 |
| 2   | 本词典 §1.2 用户 ID（`uid`）+ §6.3 业务版本前缀 + §6.2 业务表操作字段（`operator_uid` / `client_ip`） | 高   | 已与规范 v1.1.1 双向锚定，规范同步修订 |
| 3   | 《数据库设计规范》§2.5 字段命名通用规则中的**示例**（如 `is_deleted` 作 `is_` 前缀样例） | 中   | 仅作命名形态示例，不是字段推荐；本词典可在不冲突前提下细化 |
| 4   | 本词典 §1\~§9 其他字段语义规则 | 中   | 在规范未明确处填补 |
| 5   | 本词典 §10 现状冲突清单 / §13 豁免 | 低   | 域级最终落地依据 |

**实践快速记忆：**

* 看到 `version` → 乐观锁；看到 `xxx_version` → 业务版本
* `log_*` 审计表用 `operator` / `operator_ip`；业务表用 `operator_uid` / `client_ip`
* 软删除一律 `deleted_at`，永远不要 `is_deleted` 字段
* 用户主体字段一律 `uid`，永远不要 `user_id` 字段
* 余额字段推荐 `_balance` 后缀；旧表沿用 `available` / `frozen` 不强迁，但同表内须风格一致


---

## 版本记录

| 版本  | 日期  | 摘要  |
|-----|-----|-----|
| v1.0.0 | 2026-04-30 | 初版；9 大类字段 + 17 项冲突 + 3 项豁免 |
| v1.0.1 | 2026-04-30 | 与《数据库设计规范》v1.1.1 双向锚定：(1) `uid` 作为唯一用户 ID（规范同步反向修订）；(2) 裸 `version` 让位乐观锁（规范 §4.7 既定），业务版本必须前缀；(3) `is_deleted` 软化为"仅作 `is_` 前缀示例"；(4) `available` / `frozen` 视为 `available_balance` 等价；(5) 审计表 `operator` / `operator_ip` 沿用规范 §4.8；(6) 缩写白名单收紧到 5 个；(7) 新增附录 C 优先级裁决 + 5 条快速记忆 |