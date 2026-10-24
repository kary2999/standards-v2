---
title: "MySQL 数据库设计规范"
version: "2.2.0-draft"
last_modified: "2026-10-22"
---
<!-- [skill: go-team-standards · 数据库设计 · 字段命名 · ID 生成] MySQL 数据库设计规范 -->

# MySQL 数据库设计与变更规范

> 版本：V2.2.0-draft | 状态：**草案，待 Tech Lead + DBA 评审** | 适用范围：MySQL 8.0 / 8.4 LTS（InnoDB）
>
> 技术栈：Go + Gin + MySQL 8.4 + Redis + Kafka。字段语义命名见 [field-naming.md](field-naming.md)，主键 / 分布式 ID 见 [id-generation.md](id-generation.md)；**本文与二者冲突处，以本文为准**，冲突项集中登记在 §12。Redis 见 [redis.md](redis.md)，Kafka 见 [kafka.md](kafka.md)。

---

## 1. 总则

数据库是交易所系统的核心资产。所有 Schema 变更必须经过评审、版本化管理，并通过 CI Pipeline 自动执行。**禁止任何人手动在生产环境执行 DDL。**

---

## 2. 环境基线

### 2.1 版本

* 生产使用 **MySQL 8.4 LTS**（存量 8.0.x 须在 8.0 EOL 前升级）
* 禁止在生产使用 Innovation 版本 / RC / Beta
* 升级窗口提前一周公告，灰度副本验证后再滚动

### 2.2 实例级强制配置

| 参数 | 值 | 说明 |
|---|---|---|
| `default_storage_engine` | `InnoDB` | 禁 MyISAM / MEMORY |
| `character_set_server` | `utf8mb4` | **禁 `utf8`（即 utf8mb3）** |
| `collation_server` | `utf8mb4_0900_bin` | 二进制比较，区分大小写，见 §3.7 |
| `default_time_zone` | `+00:00` | **实例强制 UTC**。`DATETIME` 不带时区，`CURRENT_TIMESTAMP(6)` 按**会话**时区取值，所以必须全链路统一 UTC，且**禁止在连接 / 会话里 `SET time_zone`**（已实测：会话时区为 +08:00 时，默认值写入的是本地时间，没有任何报错） |
| `explicit_defaults_for_timestamp` | `ON` | 8.0 默认值，保持 ON，不得改动（本规范不使用 `TIMESTAMP`） |
| `transaction_isolation` | `READ-COMMITTED` | 减少间隙锁与死锁（MySQL 默认是 REPEATABLE-READ，必须改） |
| `sql_mode` | `STRICT_TRANS_TABLES,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION,ONLY_FULL_GROUP_BY` | **禁止关闭严格模式**，否则超长字符串被静默截断、漏写 NOT NULL 列被填 0 |
| `lower_case_table_names` | `0`（Linux） | 初始化时确定，之后不可改 |
| `max_execution_time` | `30000`（ms） | 仅对 SELECT 生效 |
| `innodb_lock_wait_timeout` | `3`（s） | 行锁等待上限 |
| `innodb_rollback_on_timeout` | `ON` | 锁等待超时回滚整个事务 |
| `wait_timeout` / `interactive_timeout` | `60`（s） | 防空闲连接占用 |
| `binlog_format` | `ROW` | 禁 STATEMENT / MIXED |
| `gtid_mode` / `enforce_gtid_consistency` | `ON` / `ON` | |
| `sync_binlog` / `innodb_flush_log_at_trx_commit` | `1` / `1` | 资金库双 1，不得为性能降级 |
| `slow_query_log` / `long_query_time` | `ON` / `0.2` | 阈值 200ms |
| `require_secure_transport` | `ON` | 强制 TLS |

> 写语句没有 `max_execution_time`，其超时靠 `innodb_lock_wait_timeout` + 应用层 `context.WithTimeout` 兜底。

### 2.3 连接池

* 所有服务必须通过 **ProxySQL** 接入，禁止应用直连实例
* 单服务连接池上限 **≤ 50**；公式 `pool_size = 副本数 × 单副本并发需求 + 缓冲`
* Go 侧必须显式设置，禁止用 `database/sql` 默认值：

  ```go
  sqlDB.SetMaxOpenConns(50)
  sqlDB.SetMaxIdleConns(10)
  sqlDB.SetConnMaxLifetime(30 * time.Minute) // 须小于 wait_timeout 与 ProxySQL 回收时间
  sqlDB.SetConnMaxIdleTime(5 * time.Minute)
  ```
* DSN 必带：`charset=utf8mb4&parseTime=true&loc=UTC&clientFoundRows=true&timeout=3s&readTimeout=35s&writeTimeout=5s`
  * `readTimeout` 必须**大于**服务端 `max_execution_time`（30s），让慢查询由服务端以 3024 错误明确终止，而不是客户端半途断开、服务端继续跑
  * 真正的单次调用时限由 `context.WithTimeout` 控制：OLTP 默认 ≤ 5s；批处理 / 对账 / 报表按需放宽，不得靠全局 `readTimeout` 卡死
* 禁止长事务（> 10s），必须拆分或改异步
* 禁止依赖会话变量 / 临时表 / `GET_LOCK` 跨语句存活（连接复用会错乱）

---

## 3. 命名规范

### 3.1 通用规则

* 全部 `snake_case` 小写。MySQL 在 Linux 下表名大小写敏感，混用会造成 macOS 开发 / Linux 生产不一致
* 禁止无意义缩写（白名单：`id` / `url` / `ip` / `uid` / `ts`）
* 对象名长度 **≤ 64 字符**
* 禁止使用保留字（`order` / `user` / `group` / `key` / `rank` / `condition` / `desc` / `row` / `rows` / `system` / `groups` / `lead` / `lag` 等），查询：

  ```sql
  SELECT WORD FROM INFORMATION_SCHEMA.KEYWORDS WHERE RESERVED = 1;
  ```
  **禁止靠反引号规避保留字**——改名
* 禁止特殊字符、非英文字符；禁止以 `_` 开头；禁止匈牙利前缀（`tbl_` / `col_`）
* 字段名不得包含所在表名（`users` 表用 `name`，不用 `user_name`）；作外部引用时除外

### 3.2 Database 命名

MySQL 中 Schema ≡ Database，**业务隔离的单位就是库**。

| 用途 | 命名 | 示例 |
|---|---|---|
| 业务库 | `{业务域}` | `trade` / `wallet` / `auth` / `market` / `risk` / `audit` |
| 分片库 | `{业务域}_shard_{两位序号}` | `trade_shard_00` ~ `trade_shard_15` |

* 业务域：`trade`（撮合/订单/成交）、`wallet`（资产/充提/余额）、`auth`（用户/权限/KYC）、`market`（行情）、`risk`（风控）、`audit`（审计，只追加）
* 禁止使用 `test` / `db1` / `mysql` / `sys` 等名称
* **禁止跨库 JOIN / 跨库事务**；一个服务只写自己的库，其他服务走接口或事件

### 3.3 表命名

| 前缀 | 类型 | 格式 | 示例 |
|---|---|---|---|
| 无前缀 | 核心业务表 | `{库}.{实体复数}` | `trade.orders` |
| `dim_` | 维度表 | `dim_{域}_{实体}` | `dim_user_profile` |
| `agg_` | 聚合统计表 | `agg_{域}_{指标}_{粒度}` | `agg_trade_volume_daily` |
| `cfg_` | 配置表 | `cfg_{模块}_{含义}` | `cfg_fee_rules` |
| `log_` | 操作 / 事件日志（只追加） | `log_{域}_{事件}` | `log_user_login` |
| `hist_` | 历史快照（只追加） | `hist_{域}_{实体}` | `hist_account_balance` |
| `rel_` | 多对多关系 | `rel_{A}_{B}`（字母序） | `rel_user_role` |
| `tmp_` | 临时表 | `tmp_{域}_{业务}_{时间}` | `tmp_settlement_batch_202610` |
| `v_` | 视图 | `v_{描述}` | `v_active_orders` |
| `{主表}_ext` | 垂直扩展表 | 与主表 `oid` 一对一（`oid` 同值，建唯一索引） | `users_ext` |

* MySQL 无物化视图，需要时用 `agg_*` 表 + 定时任务
* `log_` / `hist_` 禁止 UPDATE / DELETE：通过**账号权限**实现（只授 `INSERT, SELECT`），不靠约定
* `tmp_` 超过 7 天未使用须清理
* 分区名用 `p202610` / `p_max`

### 3.4 字段命名

| 类别 | 必须 | 禁止 |
|---|---|---|
| 用户主体 | `uid`（`VARCHAR(64)`） | `user_id` / `member_id` / `customer_id` |
| 多租户 | `platform_id` / `uid` / `org_id` | `tenant_id` / `brand_id` |
| 时间点（业务） | `create_time` / `update_time` / `{动作}_time`（`BIGINT UNSIGNED` 毫秒） | `gmt_create` / `ctime` / 裸 `time` / `ts` |
| 时间点（库维护） | 仅 `created_at` / `updated_at`（`DATETIME(6)`） | 其他 `xxx_at` |
| 业务时间点 | `{动作过去式}_time`：`paid_time` / `settled_time` / `expire_time`，类型 `BIGINT UNSIGNED` 毫秒 | `xxx_at`（`created_at` / `updated_at` 除外）、`xxx_timestamp` |
| 删除标记 | 无（本规范不使用软删除，见 §4.4） | `deleted_at` / `is_deleted` / `is_alive` |
| 布尔 | `is_` / `has_` / `can_` 开头 | `enabled` / `active` / `xxx_flag` |
| 金额 | `{业务}_amount`（`fee_amount`） | 裸 `amount` |
| 状态 | `xxx_status` | 裸 `state` |
| 业务版本 | `rule_version` / `config_version` | 裸 `version`（裸 `version` = 乐观锁） |
| 备注 | `remark` / `description` / `review_note` | `comment` / `memo` |
| 库内主键 | `id`（`BIGINT UNSIGNED AUTO_INCREMENT`） | `pk` / `row_id` / `seq_id` |
| 业务 ID | `oid`（`CHAR(36)`，UUIDv7） | `uuid` / `guid` / `order_no` 当主键 |
| 逻辑关联 | `{实体单数}_oid`（存对方的 `oid`，`CHAR(36)`） | `{实体}_id` 存 `oid`、`{实体复数}_oid`；**禁 `FOREIGN KEY`** |

### 3.5 索引与约束命名

| 对象 | 格式 | 示例 |
|---|---|---|
| 主键 | `PRIMARY`（固定，不可改名） | — |
| 唯一索引 | `uk_{表}_{字段...}` | `uk_users_platform_email` |
| 普通索引 | `idx_{表}_{字段...}` | `idx_orders_platform_uid_ctime` |
| Check 约束 | `ck_{表}_{含义}` | `ck_orders_positive_qty` |

* 总长 ≤ 64 字符；Check 约束需 MySQL ≥ 8.0.16 才真正生效
* 唯一约束与唯一索引在 MySQL 是同一物，统一只用 `uk_`

### 3.6 账号命名

* 每个服务独立账号，最小权限：`{服务名}_ro` / `{服务名}_rw`，如 `trade_svc_rw`
* 迁移专用账号 `{服务名}_ddl`，仅 CI Pipeline 持有，应用运行时**不得**持有 DDL 权限
* 禁止多服务共用账号；禁止应用使用 `root` / 带 `SUPER` 权限账号
* 账号绑定来源网段 / ProxySQL 地址，禁止 `'user'@'%'`

### 3.7 字符集、排序规则与注释

* 全库 `utf8mb4` + `utf8mb4_0900_bin`，**建表必须显式写出**
* 选 `_bin`：`uid` / `coin` / `symbol` / `tx_hash` 等标识必须区分大小写且无重音折叠，否则唯一索引会冲突。需要大小写不敏感的字段（邮箱）由**应用层小写归一**后存储
* 同一 JOIN / 比较的两列字符集与排序规则必须一致，否则索引失效
* 所有表和字段**必须**写 `COMMENT`（建表时内联）；枚举 / 状态字段 COMMENT 里列全取值；含义变更须同步更新 COMMENT（`MODIFY COLUMN` 须写全列定义，漏写会改掉类型）

---

## 4. 设计规范

### 4.1 必备字段

每张业务表必须包含以下字段：

```sql
`id`          BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '库内主键，自增，仅本表内部使用',
`oid`         CHAR(36)         NOT NULL COMMENT '业务 ID，UUIDv7 小写 36 位，应用层生成',
-- ... 业务字段 ...
`create_time` BIGINT UNSIGNED  NOT NULL COMMENT '创建时间，毫秒时间戳（UTC）',
`update_time` BIGINT UNSIGNED  NOT NULL COMMENT '更新时间，毫秒时间戳（UTC）',
`created_at`  DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
`updated_at`  DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '更新时间'
```

**两套时间字段的分工：**

| 字段 | 谁写 | 精度 | 用途 |
|---|---|---|---|
| `create_time` / `update_time` | **应用层**写入 | 毫秒 | 业务逻辑、游标分页、分区键、跨系统传递（API / Kafka 的毫秒时间戳） |
| `created_at` / `updated_at` | **数据库**维护 | 微秒（`DATETIME(6)`） | 运维排查、人工查库可读、DBA 对账 |

规则：

* `create_time` / `update_time` 无数据库默认值，**由应用层 `time.Now().UnixMilli()` 写入**；漏写在严格模式下直接报错（这是有意为之）
* **每条 UPDATE 都必须同时写 `update_time`**，含手写 SQL、批量更新、回填脚本。漏写 `update_time` 列为 Code Review 阻断项。`updated_at` 由数据库自动维护，不需要写
* 禁止应用层写 `created_at` / `updated_at`
* 两套时间来自不同时钟（应用服务器 vs 数据库服务器），可能有毫秒到秒级偏差。**业务以 `create_time` / `update_time` 为准**；`created_at` / `updated_at` 仅作运维参考，业务代码不得读取
* 本规范**不设** `created_by` / `updated_by` / `deleted_at`；操作人追溯依赖 `log_*` 审计表（§4.9）与 binlog

**`created_at` / `updated_at` 的类型规则（`DATETIME(6)`）：**

* 必须写成 `DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)`；`updated_at` 另加 `ON UPDATE CURRENT_TIMESTAMP(6)`。**`DEFAULT` / `ON UPDATE` 的精度必须与列精度一致**：`DATETIME(6)` 配不带 `(6)` 的 `CURRENT_TIMESTAMP` 会直接报错（已实测：`ON UPDATE CURRENT_TIMESTAMP` 报 1294，`DEFAULT CURRENT_TIMESTAMP` 报 1067）
* **`DATETIME` 不存时区**，取值依赖会话时区：实例 `default_time_zone` 必须为 `+00:00`，应用 DSN 带 `loc=UTC`，任何连接都不得 `SET time_zone`。会话时区一旦偏离 UTC，写入的就是本地时间且无任何提示（已实测）
* `ON UPDATE` 只在行内容**真正发生变化**时才刷新 `updated_at`（已实测：同值 `UPDATE` 不改变它），所以不能用它判断“有没有被更新过”
* 取值范围 `1000-01-01` ~ `9999-12-31`，没有 `TIMESTAMP` 的 2038 年问题（已实测可写入 `9999-12-31 23:59:59.999999`）
* **全库禁用 `TIMESTAMP` 类型**；未来时间（到期、计划执行）仍用 `BIGINT` 毫秒（§4.6）

### 4.2 设计原则

* **禁止 `FOREIGN KEY`**，关联由应用层维护
* 除 `TEXT` / `JSON` 外，业务字段一律 `NOT NULL` 并给默认值。理由：唯一索引中多个 NULL 不冲突、`NOT IN` 判断出错、索引变大
* **不使用软删除，允许物理 `DELETE`**（见 §4.4）
* 单表字段 ≤ **30**，超出垂直拆分为 `_ext`
* 单表行数预警线 **2000 万** / 容量 **20 GB**，超限须规划分区或分片（§5）
* 大字段（`TEXT` / `JSON`）不与高频读写的热字段放同一张表
* 禁止存储过程、触发器、事件调度器（`EVENT`）、自定义函数——逻辑放 Go 代码，可测试、可灰度、可回滚

### 4.3 主键与业务 ID：自增 `id` + UUIDv7 `oid`

每张业务表**同时**有两个 ID（完整规则见 [id-generation.md](id-generation.md)）：

| 字段 | 类型 | 谁生成 | 角色 |
|---|---|---|---|
| `id` | `BIGINT UNSIGNED NOT NULL AUTO_INCREMENT` | 数据库 | 库内物理主键（聚簇索引），**只在本表内部使用** |
| `oid` | `CHAR(36) NOT NULL` + `UNIQUE KEY uk_{表}_oid` | 应用层 UUIDv7，写入前已知 | **业务 ID**：订单号、API、Kafka、Redis、日志、跨表 / 跨服务引用 |

**`id` 的边界**
* 允许：作聚簇主键；本表内稳定排序 / 游标 tie-breaker；运维按 `id` 增量扫描
* **禁止**：出现在 API / Kafka / Redis Key / 前端；被其他表或服务引用为逻辑外键；当业务编号、订单号、幂等键；业务读取 `LAST_INSERT_ID()`（业务要的标识是插入前就生成的 `oid`）；假设连续（回滚、批量插入、失败重试都会留空洞）；跨库 / 跨分片唯一（各片会重复）
* 类型必须是 `BIGINT UNSIGNED`，不用 `INT`

**`oid` 的规则**
* 由 common-lib `idgen` 生成，禁止业务代码裸调 `uuid.NewV7()` / `uuid.New()`；**禁止** UUIDv4（无序）、UUIDv1（含 MAC）
* **36 位小写带连字符**。库默认排序规则 `utf8mb4_0900_bin` 区分大小写，所以必须小写归一后再写入、再查询
* 引用其他实体用 `{实体单数}_oid`（`order_oid`），类型 `CHAR(36)`；不用 `{实体}_id` 存 `oid`，避免和对方表的自增 `id` 混淆
* 需要“人类可读订单号”时另设展示号字段（如 `order_ref`），不复用 `oid` / `id`

#### `oid` 必须是 `CHAR(36)`（硬性要求，不得改成 `BINARY(16)`）

`oid` 要求**肉眼可读**：命令行、日志、报表、工单里直接看到、复制、粘贴就能查，不需要 `BIN_TO_UUID()` 之类的转换。所以：

* 类型固定 `CHAR(36)`，存 36 位小写带连字符字符串（如 `01a1639e-afc8-73ad-821d-06201057ccd5`）
* **禁止**把 `oid` 改为 `BINARY(16)` / `VARBINARY` / 其他二进制或压缩形式，任何表、任何数据量都不例外
* 空间代价可以接受：`oid` 不是主键，不会被复制进其他二级索引，只在 `uk_*_oid` 这一个唯一索引里出现一次（每条记录 36 字节加 8 字节 `id`），主键 `id` 仍保持 8 字节

查询示例（直接用字符串，无需转换）：

```sql
SELECT id, oid, symbol FROM trade.orders WHERE oid = '01a1639e-afc8-73ad-821d-06201057ccd5';
```

#### 为什么主键仍选自增 `BIGINT`

* 追加写入、聚簇索引紧凑；每条二级索引只多带 **8 字节**主键（`CHAR(36)` 主键会是 36 字节）
* 业务身份已由 `oid` 承担，`id` 不对外，不会暴露业务量

#### 自增 `id` 的注意事项

* 单写主库；`innodb_autoinc_lock_mode` 保持 8.x 默认（`2`，配合 `binlog_format=ROW`）
* 8.0+ 自增计数器持久化，重启不回退；但**回滚、`INSERT IGNORE` 失败、批量插入仍会留空洞**，属正常
* 分区表主键须为 `(id, create_time)`（§5.1）
* 分库分表后各片 `id` 会重复，跨片身份只认 `oid`

#### 不用 Snowflake

| 问题 | 说明 |
|---|---|
| 时间戳位数有寿命 | 标准 Snowflake 的 41 位毫秒时间戳只够约 **69.7 年**（从自定义 epoch 起算），耗尽后 ID 溢出；改位宽 = 全库 ID 格式不兼容 |
| 63 位超出 JS 安全整数 | BIGINT 以 JSON 数字下发前端会静默丢精度，必须全链路改字符串 |
| 时钟回拨 / worker-id | 依赖时钟与 worker-id 分配，配置重复或时钟异常会**静默**发出重复 ID |
| UUIDv7 对照 | 48 位毫秒时间戳约 8920 年（公元 10889 年）才耗尽；无需 worker-id；同进程内严格单调（`google/uuid` v1.6.0 加锁保证）；跨进程撞号概率极低，且 `uk_*_oid` 兜底 |

**撞号处理**：INSERT 遇 MySQL 错误 1062 时**看错误信息里的索引名**——命中 `uk_{表}_oid` 才重新生成 `oid` 重试（最多 3 次）；命中业务唯一键是幂等重复，按幂等语义处理，**不得**换 `oid` 重试。业务幂等靠 `client_order_id` / `idempotency_key`，不靠 `oid` 或 `id` 的唯一性。

### 4.4 删除策略

* 本规范**不使用软删除**：表中没有 `deleted_at` / `is_deleted` / `is_alive`，唯一索引直接按业务键建立，无需为“已删除记录”特殊处理
* 允许对业务表物理 `DELETE`，但必须满足：
  * `DELETE` 必须带 `WHERE` 且命中索引；批量删除遵守 §7.2 分批规则（单批 ≤ 1000 行、批间 sleep）
  * 涉及资金、订单、成交、账务的数据，删除前须先写入对应 `hist_*` 归档表或 `log_*` 审计表，不得直接丢弃
  * 物理删除**不可逆**，恢复只能靠 binlog / 备份（§8.3），因此生产环境的删除须走评审，不得在运维会话里手敲
* `log_*` / `hist_*` 表仍禁止 UPDATE / DELETE（账号只授 `INSERT, SELECT`）；其过期数据只能通过分区 `DROP`（§9.3）清理
* 需要“可恢复的删除”语义的业务，用状态字段（如 `xxx_status` = 已作废）表达，而不是删除标记

### 4.5 索引策略

* 逻辑关联字段（`uid` / `order_id` 等）必须建索引
* 单表索引 ≤ **8** 个；组合索引遵循最左前缀；区分度低的字段（`status`）不单独建索引
* **多租户表索引前缀必须含 `platform_id`**
* InnoDB 二级索引叶子节点存主键，主键越宽索引越大——这是主键选 8 字节自增 `BIGINT`、而把 36 字节 `oid` 放进单独唯一索引的原因
* `utf8mb4` 下单列 `VARCHAR(n)` 索引按 `4n` 字节计，索引键上限 3072 字节；必要时用前缀索引
* 范围查询列放组合索引**最后**（范围之后的列无法使用索引）
* 利用覆盖索引避免回表；8.0 支持降序索引 `(create_time DESC)` 与函数索引
* **禁止隐式类型转换**：`WHERE uid = 12345`（uid 是 VARCHAR）会全表扫描；`oid` 要传小写 36 位字符串（`CHAR(36)` 列），不得传大写或去掉连字符的形式
* 在线建索引必须显式声明算法：

  ```sql
  ALTER TABLE trade.orders
    ADD INDEX idx_orders_platform_uid_ctime (platform_id, uid, create_time DESC),
    ALGORITHM=INPLACE, LOCK=NONE;
  ```
  若因算法不支持报错，**不要去掉 ALGORITHM 子句强跑**（会静默退化成锁表），转走 gh-ost（§6.3）
* 删除索引前先 `ALTER INDEX ... INVISIBLE` 观察一周，确认无慢查询再 `DROP`

### 4.6 字段类型约束

#### 时间

* **业务时间点一律 `BIGINT UNSIGNED` 毫秒时间戳（UTC）**，字段名 `{动作过去式}_time`（`paid_time` / `settled_time`），计划时间 `expire_time` / `scheduled_time`；与 REST JSON、Kafka 的毫秒时间字段一一对应，无需转换
* 纯日期用 `DATE`，后缀 `_date`
* **全库禁用 `TIMESTAMP`**（2038 上限、依赖会话时区转换）；库维护的 `created_at` / `updated_at` 用 `DATETIME(6)`，其余业务时间点一律 `BIGINT UNSIGNED` 毫秒
* 命名细则见 [field-naming.md](field-naming.md) §2

#### 金额 / 数量

| 场景 | 类型 | 例 |
|---|---|---|
| 加密货币数量 / 余额 / 价格 / 矿工费 / 权利金 | `DECIMAL(38,18)` | `available_balance` `filled_qty` `mark_price` |
| 法币 / U 卡等价金额 | `DECIMAL(28,8)` | `cash_amount` |
| 费率 / 利率 / APY | `DECIMAL(10,8)` | `maker_rate` `apy` |
| 计数 | `BIGINT UNSIGNED` / `INT UNSIGNED` | `retry_count` |

* **禁 `FLOAT` / `DOUBLE`**；禁用整型代替小数（除非单位明确为最小不可分单位，如 satoshi，且字段名标明）
* Go 侧映射 `shopspring/decimal`，**扫描时不得经过 `float64`**
* 余额类字段加 `CHECK (available_balance >= 0)` 作最后一道防线
* 类型变更需 Tech Lead + DBA 双审批

#### 小值域（枚举 / 状态 / 布尔）

* 值域 ≤ 20 的字段一律 `TINYINT UNSIGNED`，**禁** `ENUM`、`BOOLEAN`、`BIT`、`VARCHAR`（`ENUM` 增删取值需 DDL，且按内部序号排序）
* 编码从 `1` 开始，`0` 保留“未知 / 默认”；布尔用 `1=是 2=否 0=未知`，字段名仍以 `is_` / `has_` / `can_` 开头
* 常量映射集中在各域 `-lib` 包，COMMENT 里同步列出

#### 字符串

* 固定长度标识用 `CHAR(n)`：`country_code CHAR(2)`、`currency CHAR(3)`
* 变长用 `VARCHAR(n)`，n 按真实最大需求给，不要一律 255（`utf8mb4` 下 n 是字符数）
* `TEXT` 仅限备注、描述、日志内容等无查询条件字段；不得作 WHERE 条件、不得建索引、不得设默认值
* 禁 `LONGTEXT` / `LONGBLOB`；文件走对象存储，库里只存 key
* 常用长度：`uid VARCHAR(64)`、`platform_id VARCHAR(32)`、`org_id VARCHAR(64)`、`coin VARCHAR(20)`、`symbol VARCHAR(40)`、`tx_hash VARCHAR(128)`、`trace_id VARCHAR(32)`、`idempotency_key VARCHAR(128)`

#### IP 地址

无 `INET` 类型，存 `VARBINARY(16)`，读写用 `INET6_ATON()` / `INET6_NTOA()`（兼容 IPv6）。禁用 `INT UNSIGNED` + `INET_ATON`。

#### JSON

* 仅用于**非核心交易数据**的扩展属性（偏好、KYC 附加信息、审计快照）
* 禁止存金额、状态、时间等交易核心字段
* 禁止以 JSON 内部字段作主要查询条件；确需时用**生成列 + 索引**提取：

  ```sql
  `risk_level` TINYINT UNSIGNED GENERATED ALWAYS AS (JSON_EXTRACT(`extra`, '$.risk_level')) STORED,
  KEY `idx_xxx_risk_level` (`risk_level`)
  ```
* 单字段 ≤ **64 KB**；更新用 `JSON_SET` 做局部更新，整列覆盖会放大 binlog

### 4.7 多租户字段

| 字段 | 类型 | 可空 |
|---|---|---|
| `platform_id` | `VARCHAR(32)` | NOT NULL |
| `uid` | `VARCHAR(64)` | NOT NULL（**禁 `BIGINT`**） |
| `org_id` | `VARCHAR(64)` | NULL（非机构账户为空） |

* 索引前缀至少含 `platform_id`
* **MySQL 无行级安全**，租户隔离靠：
  1. 所有查询经 Repo 层统一注入 `platform_id` 条件，禁止业务代码手写无租户条件的 SQL
  2. 报表 / 运营等跨租户角色只授予按 `platform_id` 过滤的**视图**，不授表权限
  3. Code Review 与 SQL Lint 把“多租户表缺 `platform_id` 条件”列为阻断项
* `platform_id` 全链路透传（HTTP 请求头 / Kafka header / Redis key / 日志字段）

### 4.8 乐观锁

对并发更新的核心实体（余额、订单状态）必须启用：

```sql
UPDATE trade.orders
   SET order_status = 3, version = version + 1, update_time = ?
 WHERE oid = ? AND version = ?;
-- 必须检查 RowsAffected：=0 表示版本冲突，业务层重试或失败
```

* 字段 `version INT UNSIGNED NOT NULL DEFAULT 0`；裸 `version` **只**用于乐观锁
* **MySQL 陷阱**：`RowsAffected` 默认返回“实际被修改的行数”，值没变就是 0。DSN 必须带 `clientFoundRows=true`（返回匹配行数，已在 8.4.11 实测：同值 UPDATE 在该参数开启时返回 1、关闭时返回 0）。
  * 带 `version = version + 1` 的乐观锁更新本身一定会改动行，不受此影响
  * 受影响的是**可能不改变任何值**的更新（如把状态置为已有的同值的幂等流转、手写 SQL 不带 `update_time`）：没有该参数会被误判为“未命中 / 冲突”
* 余额扣减、订单状态流转必须“乐观锁 + 幂等键”双保险
* 禁止用 `SELECT ... FOR UPDATE` 做跨行业务编排

### 4.9 审计 / 日志表标准字段

`log_*` 表除 §4.1 外还必须包含：

| 字段 | 类型 |
|---|---|
| `trace_id` | `VARCHAR(32) NOT NULL` |
| `operator` | `VARCHAR(64)` |
| `operator_ip` | `VARBINARY(16)` |
| `action` | `VARCHAR(64) NOT NULL` |
| `target_type` | `VARCHAR(64) NOT NULL` |
| `target_id` | `VARCHAR(64) NOT NULL` |
| `before_snapshot` / `after_snapshot` / `extra` | `JSON` |

* 必须按月 / 季度分区（§5.1）；账号权限仅 `INSERT, SELECT`
* 变更前后快照用 `before_snapshot` / `after_snapshot`，因为 `before` 是 MySQL 保留字（登记见 §12）

### 4.10 CREATE TABLE 标准模板

```sql
-- [skill: go-team-standards · 数据库设计 · 字段命名] 现货订单表
-- migrations/000123_trade_create_orders.up.sql
CREATE TABLE `trade`.`orders` (
    `id`               BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '库内主键，自增，仅本表内部使用',
    `oid`              CHAR(36)         NOT NULL COMMENT '业务 ID，UUIDv7 小写 36 位，应用层 idgen 生成',

    -- 多租户必备
    `platform_id`      VARCHAR(32)      NOT NULL COMMENT '平台 ID',
    `uid`              VARCHAR(64)      NOT NULL COMMENT '用户 ID',

    -- 业务字段
    `client_order_id`  VARCHAR(64)      NOT NULL COMMENT '客户端订单号，幂等键',
    `symbol`           VARCHAR(40)      NOT NULL COMMENT '交易对 BASE-QUOTE',
    `order_side`       TINYINT UNSIGNED NOT NULL COMMENT '方向：1=买 2=卖',
    `order_type`       TINYINT UNSIGNED NOT NULL COMMENT '类型：1=限价 2=市价',
    `price`            DECIMAL(38,18)   NOT NULL COMMENT '委托价',
    `quantity`         DECIMAL(38,18)   NOT NULL COMMENT '委托数量',
    `filled_qty`       DECIMAL(38,18)   NOT NULL DEFAULT 0 COMMENT '已成交数量',
    `order_status`     TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态：1=待撮合 2=部分成交 3=完全成交 4=已取消 5=已过期',

    -- 并发控制
    `version`          INT UNSIGNED     NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',

    -- 必备时间与审计字段
    `create_time`      BIGINT UNSIGNED  NOT NULL COMMENT '创建时间，毫秒时间戳（UTC），应用层写入',
    `update_time`      BIGINT UNSIGNED  NOT NULL COMMENT '更新时间，毫秒时间戳（UTC），应用层每次更新必写',
    `created_at`       DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) COMMENT '创建时间',
    `updated_at`       DATETIME(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6) COMMENT '更新时间',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_orders_oid` (`oid`),
    UNIQUE KEY `uk_orders_platform_uid_client_order` (`platform_id`, `uid`, `client_order_id`),
    KEY `idx_orders_platform_uid_ctime` (`platform_id`, `uid`, `create_time` DESC),
    KEY `idx_orders_platform_symbol_status` (`platform_id`, `symbol`, `order_status`),
    CONSTRAINT `ck_orders_positive_qty` CHECK (`quantity` > 0),
    CONSTRAINT `ck_orders_valid_side`   CHECK (`order_side` IN (1, 2)),
    CONSTRAINT `ck_orders_filled_qty`   CHECK (`filled_qty` >= 0)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_0900_bin
  ROW_FORMAT=DYNAMIC
  COMMENT='现货订单表';
```

`down.sql`：

```sql
DROP TABLE IF EXISTS `trade`.`orders`;
```

> 权限不在建表 Migration 里 `GRANT`，统一由 DBA 账号管理流程下发，避免 CI 账号持有 `GRANT OPTION`。

---

## 5. 分区与分片

### 5.1 分区表

满足任一条件**必须**规划分区：

| 触发条件 | 方式 |
|---|---|
| 预计行数 > 2000 万 / 容量 > 20 GB | `RANGE`（按时间） |
| 日志 / 审计 / 流水（`log_*` / `hist_*`） | `RANGE`（按月 / 季） |
| 多租户强隔离，租户数少且稳定 | `LIST`（按 `platform_id`） |

**MySQL 分区硬约束：**

* **分区键必须包含在表的每一个唯一键（含主键）里**。分区表主键须为 `(id, create_time)`（自增列必须是索引的第一列，所以 `id` 在前），所有 `UNIQUE KEY` 也要带 `create_time`，包括 `uk_*_oid`——这会使 `oid` 与幂等键的“全局唯一”**减弱为“同分区内唯一”**，设计时必须评估；`oid` 的唯一性在分区表上主要靠 UUIDv7 本身的碰撞概率保证，不要依赖库来兜底
* **分区键用 `create_time`（BIGINT 毫秒）**，不要用 `created_at`：业务以 `create_time` 为准，口径要与分区边界一致；`created_at` 是库维护的运维字段。`BIGINT` 直接 `RANGE` 最简单可靠
* 分区数 ≤ **1024**，建议单表 ≤ 100
* 查询必须带分区键条件，否则全分区扫描；DBR 用 `EXPLAIN` 看 `partitions` 列校验

```sql
PARTITION BY RANGE (`create_time`) (
    PARTITION p202610 VALUES LESS THAN (1793491200000),  -- 2026-11-01 00:00:00 UTC
    PARTITION p202611 VALUES LESS THAN (1796083200000),  -- 2026-12-01 00:00:00 UTC
    PARTITION p_max   VALUES LESS THAN MAXVALUE
);
```

* 新分区由定时任务**提前至少 3 个月**创建（`REORGANIZE PARTITION` 拆分 `p_max`），禁止手工维护；过期分区归档后 `DROP PARTITION`（秒级，远快于 `DELETE`）
* 保留期：核心交易 2 年，日志 / 审计 6 个月

### 5.2 分库分表

阈值参考：单实例容量 > 2 TB 或写入 > 20K TPS。

* 分片键优先级：`platform_id` > `uid` > 业务核心 ID
* **禁止跨片 JOIN**；跨片聚合走数仓 / CQRS 只读视图
* 路由由中间件或应用层统一封装，业务代码禁止拼接分片库名
* 各片 Schema 完全一致；跨片事务禁用，改用 Saga / 本地消息表
* 按 `uid` 哈希取模分片；**不要**按 UUID 时间前缀分，否则写入全部打到最新分片（热点）

---

## 6. Migration 管理

### 6.1 工具与命名

* 使用 **`goose`**；文件目录 `migrations/`，命名 `{6位序号}_{库}_{描述}.up.sql` / `.down.sql`
* 一个 Migration **只改一个库**；MySQL 的 DDL 隐式提交、**无法回滚半成品**，一个文件内尽量只放一条 DDL

### 6.2 变更规则

* 每个变更独立文件；**禁止修改已发布的 Migration**
* 必须同时提供 `up` / `down`
* 数据回填独立 Migration，与 DDL 分开，且分批执行（§7.2）
* 禁止一次 Migration 同时修改多张无关表
* DDL 须幂等或失败后可人工续跑，Migration 头部写明失败后的恢复步骤

### 6.3 在线 DDL

| 操作 | 方式 |
|---|---|
| 加列（末尾，可空或带默认值） | `ALGORITHM=INSTANT`（秒级、不锁表）。同一表最多 64 次 INSTANT 变更，之后需重建 |
| 加 / 删索引 | `ALGORITHM=INPLACE, LOCK=NONE` |
| 改列类型 / 改字符集 / 改主键 / 加分区 | 触发表重建，**> 100 万行必须走 `gh-ost` 或 `pt-online-schema-change`** |
| 改列 `COMMENT` | `INSTANT`，但 `MODIFY` 须写全定义 |

* 所有在线 DDL 必须**显式写出** `ALGORITHM` 与 `LOCK`：不支持时报错而非静默锁表
* 大表变更选低峰期，提前确认从库延迟监控与磁盘空间（重建需额外约等于表大小的空间）
* 禁止在高峰期对核心表执行 `OPTIMIZE TABLE` / `ALTER ... ENGINE=InnoDB`

### 6.4 审批流程

| 变更类型 | 审批 |
|---|---|
| 新增表 | Tech Lead |
| 新增字段 / 索引 | 同组开发者 Review |
| 修改字段类型 | Tech Lead + DBA |
| 删除表 / 字段 | Tech Lead + DBA + 产品确认 |
| 大表变更（> 100 万行） | DBA 专项评审 + gh-ost 方案 |

### 6.5 设计评审（DBR）Checklist

**命名与结构**
- [ ] 库 / 表 / 字段 / 索引名符合 §3，无保留字
- [ ] 含 §4.1 全部必备字段：`id`（`BIGINT UNSIGNED AUTO_INCREMENT`）、`oid`（`CHAR(36)` + `uk_*_oid`）、`create_time`、`update_time`、`created_at`、`updated_at`
- [ ] `created_at` / `updated_at` 的定义与 §4.1 一字不差
- [ ] 多租户表含 `platform_id` / `uid`，索引前缀正确
- [ ] 所有表 / 字段有 `COMMENT`，枚举列全取值
- [ ] 显式声明 `ENGINE=InnoDB` / `utf8mb4` / `utf8mb4_0900_bin`

**字段与类型**
- [ ] 金额 / 数量 / 价格按 §4.6 区分 `(38,18)` 与 `(28,8)`，无 `FLOAT/DOUBLE`
- [ ] 全库无 `TIMESTAMP`；`created_at` / `updated_at` 为 `DATETIME(6)` 且 `DEFAULT` / `ON UPDATE` 均为 `CURRENT_TIMESTAMP(6)`；未来时间（到期、计划执行）用 `BIGINT` 毫秒
- [ ] 枚举 / 布尔均 `TINYINT UNSIGNED`，无 `ENUM` / `BOOLEAN`
- [ ] 业务字段 `NOT NULL` 且有默认值；字段数 ≤ 30
- [ ] 自增 `id` 未出现在 API / Kafka / Redis Key，未被其他表引用；跨表引用字段是 `{实体}_oid`；业务代码不读 `LAST_INSERT_ID()`

**索引与性能**
- [ ] 索引 ≤ 8，符合最左前缀，无冗余
- [ ] 无 `deleted_at` / `is_deleted` 等软删除字段；涉及物理删除的路径已按 §4.4 评审
- [ ] `EXPLAIN` 验证核心查询，无全表扫描 / 隐式类型转换
- [ ] 超 2000 万行已规划分区（§5.1），分区键 `create_time` 已纳入所有唯一键

**并发与安全**
- [ ] 并发更新实体已用乐观锁，DSN 已开 `clientFoundRows=true`
- [ ] 所有 UPDATE 路径（含手写 SQL / 批量）均写 `update_time`
- [ ] 敏感字段加密（§8.1）
- [ ] 账号权限最小化；`log_*` / `hist_*` 无 UPDATE / DELETE 权限
- [ ] 无 `FOREIGN KEY`、无触发器 / 存储过程 / EVENT

**容量与运维**
- [ ] 1 年 / 3 年容量已估算；归档策略已规划（§9.3）
- [ ] 监控接入（慢查询、容量、TPS、复制延迟）

### 6.6 压测与容量估算

* **容量估算**：单行大小 × 预估行数（含增长），记入 Migration PR
* **写入压测**：按预估 TPS 的 3 倍压，确认无锁等待、无从库延迟飙升
* **查询压测**：核心查询 P99 ≤ **100 ms**，慢查询阈值 **200 ms**
* 压测报告归档 DBR 工单，不达标禁止合入

---

## 7. 查询规范

### 7.1 基础要求

* 禁止 `SELECT *`；禁止循环内查询（N+1）
* 大数据分页用**游标分页**，禁止大 `OFFSET`
* 事务范围最小化
* 禁止 `ORDER BY RAND()`
* 禁止 `LIKE '%xxx'` / `LIKE '%xxx%'` 用于大表；全文检索走 ElasticSearch
* 禁止在 `WHERE` 列上做函数 / 运算（`WHERE DATE(created_at) = ...`），改为范围 `create_time >= ? AND create_time < ?`
* `IN` 列表 ≤ **200** 个；`JOIN` ≤ **3** 张表，被驱动表关联字段必须有索引、类型与排序规则一致
* 禁止 `INSERT` 不写列名；禁止 `REPLACE INTO`（本质是删后插，产生间隙锁与副作用）
* 禁止在含多个唯一键的表上用 `INSERT ... ON DUPLICATE KEY UPDATE`（命中哪个唯一键不确定）

### 7.2 批量操作

* 批量写入单批 ≤ **1000** 行；批量更新 / 删除必须分批，批间 `sleep` 控制复制延迟
* 分批 `DELETE` / `UPDATE` 必须 `ORDER BY` 索引列 `LIMIT n`，避免随机扫描与大事务
* 幂等写入：`INSERT ... ON DUPLICATE KEY UPDATE`（仅单唯一键表）或 `INSERT IGNORE`（会吞掉其他错误，**慎用**，需注明理由）
* 批量更新同样必须带 `update_time`

### 7.3 游标分页

```sql
-- ✅ 以 create_time 为游标，自增 id 仅作同毫秒稳定性 tie-breaker（游标对外编码时用 oid 或 create_time，不暴露 id）
SELECT id, oid, symbol, create_time
  FROM trade.orders
 WHERE platform_id = ? AND uid = ?
   AND (create_time < ? OR (create_time = ? AND id < ?))
 ORDER BY create_time DESC, id DESC
 LIMIT 50;

-- ❌ OFFSET 大翻页
SELECT ... ORDER BY create_time LIMIT 50 OFFSET 100000;
```

游标里的 `id` 只在服务内部使用，返回给前端的游标令牌须不透明编码（如 base64），不得直接暴露自增 `id`。

### 7.4 事务与锁

* 实例默认 `READ COMMITTED`（§2.2）；对账 / 扣减等需可重复读的场景，在事务内显式 `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ` 并评估间隙锁风险；禁止 `SERIALIZABLE`
* RR 下范围条件的 `SELECT ... FOR UPDATE` 会加间隙锁 / 临键锁，极易死锁与插入阻塞——这是默认用 RC 的主要原因
* 加锁必须命中索引：`FOR UPDATE` 未命中索引会**升级为表锁**
* 跨表加锁顺序固定（字典序表名或业务优先级）
* 死锁需重试：捕获错误 `1213 (ER_LOCK_DEADLOCK)` 与 `1205 (ER_LOCK_WAIT_TIMEOUT)`，**整个事务**重试，最多 3 次；其余错误不得盲目重试
* 长事务（> 10s）告警，> 30s 自动 kill；死锁频次 > 10 次/小时触发排障

### 7.5 慢查询

* 阈值 **200 ms**；开启 `slow_query_log` 与 `performance_schema`，每日导出 Top 50（`pt-query-digest` 或 `events_statements_summary_by_digest`）
* 新慢查询首次出现 24 小时内分配 Owner
* 优化闭环：定位 → `EXPLAIN ANALYZE` → 索引 / 重写 → 压测 → 上线 → 复核
* 告警：单 SQL TPS > 1000 且 P99 > 500 ms 立即告警

---

## 8. 数据安全

### 8.1 敏感字段加密

* 手机号、证件号、银行卡号、API Secret：**应用层 AES-256-GCM**，密钥由 KMS 管理
* **不使用** MySQL 的 `AES_ENCRYPT()`（默认 ECB 模式，密钥会进入 binlog 与慢日志）
* 密文字段后缀 `_enc`（`VARBINARY`），检索用哈希字段后缀 `_hash`（`CHAR(64)`，加盐 SHA-256）
* 禁止硬编码密钥；日志 / 慢日志中不得出现明文

### 8.2 权限

* 账号按服务粒度最小授权，读 / 写 / DDL 分离（§3.6）
* 生产库禁止开发直连，走堡垒机 + 审计
* DDL 仅 CI Pipeline 执行，人工执行需双人复核
* 启用 `validate_password` 组件；账号使用 `caching_sha2_password`，禁用 `mysql_native_password`

### 8.3 备份与恢复

* 全量每日（`xtrabackup` / `mysqlbackup`）+ binlog 持续归档，保留 **30 天**
* 每月至少一次灾备演练，RPO ≤ 1 小时，RTO ≤ 4 小时
* 备份加密、异地对象存储；**必须验证可还原**，未经演练的备份视为不存在

### 8.4 审计

* 启用审计插件，记录所有 DDL 与敏感表 DML
* 审计日志保留 ≥ 6 个月，写入 `audit` 库并只追加
* 高敏操作（`DROP`、无 `WHERE` 的 `UPDATE/DELETE`、批量修改）实时告警
* 运维账号会话开启 `sql_safe_updates`

### 8.5 数据脱敏

生产数据下发到测试 / 开发环境必须脱敏：

| 字段类型 | 脱敏策略 |
|---|---|
| 手机号 / 邮箱 | 保留首尾，中间星号 / 按 hash 重写 |
| 身份证 / 银行卡 | 全量随机重写，保留格式 |
| 姓名 / 地址 | 替换为同长度随机字符串 |
| 余额 / 金额 | 乘以随机扰动系数（0.5 ~ 1.5），保留精度 |
| `platform_id` / `uid` | 全局重映射，保持关联一致 |
| API Key / Secret | 清空或替换测试固定值 |

脱敏经 DBA 的 ETL 统一执行；禁止开发手工 `mysqldump` 生产库；开发测试环境禁止导入未脱敏数据。

---

## 9. 运维规范

### 9.1 表维护

* 禁止业务高峰执行 `OPTIMIZE TABLE`；碎片率 > 30% 由 DBA 用 gh-ost 重建
* 大批量写入后 `ANALYZE TABLE`；关键列可建直方图：`ANALYZE TABLE t UPDATE HISTOGRAM ON col`
* 监控 `innodb_buffer_pool` 命中率、`Innodb_row_lock_waits`、历史列表长度（`trx_rseg_history_len`，过长说明长事务阻塞 purge）

### 9.2 读写分离

* 写与强一致读走主库；弱一致读（报表、行情、历史）走只读副本
* 在 Repo 层显式标注读写路由，禁止隐式依赖
* **复制延迟阈值 1s**，超阈值告警并自动摘除副本流量
* 刚写完立刻读（read-your-writes）必须走主库
* 账单 / 资金对账类查询即使是读也走主库
* 副本不承载 > 30s 的分析查询，走数仓 / OLAP
* 资金库使用半同步复制（`rpl_semi_sync_*`）或 MGR，不得纯异步

### 9.3 归档与冷热分离

* 在线保留窗口：核心交易 2 年，日志 / 审计 6 个月
* 流程：定时任务将过期分区 `EXCHANGE PARTITION` 切出 → 写入 `hist_*` 或对象存储（Parquet）→ 7 天无查询依赖后 `DROP PARTITION`
* 归档表保留与原表一致的 Schema
* 对象存储生命周期：3 个月热 → 2 年温 → 7 年冷

---

## 10. Kafka 与 Redis

* Redis Key 命名、缓存与 MySQL 的一致性、分布式锁：见 [redis.md](redis.md)
* Kafka Topic 命名、消息格式、幂等消费、DLQ、与 MySQL 的本地消息表（Outbox）：见 [kafka.md](kafka.md)

---

## 11. Go 侧落地约定

| 项 | 约定 |
|---|---|
| 驱动 / ORM | 公共库 `gin-go-common-lib/storage/db`：`NewMySQL(cfg, log)`（GORM + `gorm.io/driver/mysql`，已带慢查询日志与连接池）；禁止拼接 SQL 字符串，一律占位符 |
| DSN | `charset=utf8mb4&parseTime=true&loc=UTC&clientFoundRows=true&timeout=3s&readTimeout=35s&writeTimeout=5s` |
| 超时 | 所有 DB 调用必须传 `ctx` 且带超时，`db.WithContext(ctx)` |
| 金额 | `decimal.Decimal`，禁止 `float64` |
| 毫秒时间 | 模型字段 `CreateTime int64 \`gorm:"autoCreateTime:milli"\``、`UpdateTime int64 \`gorm:"autoUpdateTime:milli"\``；**手写 SQL / 批量更新不会触发该 tag，须手动写 `update_time`** |
| 数据库时间 | `created_at` / `updated_at` 模型中标记 `<-:false`（只读），应用层不写 |
| 主键与业务 ID | 模型 `ID uint64`（`gorm:"primaryKey;autoIncrement"`，不进 DTO / 事件 / 缓存 Key）与 `OID string`（写入前由 `idgen` 生成的 36 位小写 UUIDv7）；业务代码只传递 `OID`，不读回 `ID` |
| 内存中大量暂存 ID | 用 `uuid.UUID`（`[16]byte`，16 字节，可作 map key），不要存 36 字符 `string`（实测每个约 64 字节）；边界处 `uuid.Parse` / `.String()` 转换 |
| `oid` 参数 | 直接传 36 位小写字符串；进出库前用 `NormalizeOID` 归一为小写（`utf8mb4_0900_bin` 区分大小写） |
| 删除 | 模型**不得**含 `gorm.DeletedAt`（否则 GORM 会把 `Delete` 悄悄变成软删除）；物理删除须遵守 §4.4 |
| 错误 | 唯一键冲突（1062）、死锁（1213）、锁超时（1205）转换为 `errors.AppError`（见 [error-codes.md](error-codes.md)），不透传驱动原始错误 |
| 日志 | 慢 SQL 与错误用 `slog`，带 `trace_id`；SQL 参数中的敏感字段脱敏 |
| 事务 | `db.Transaction(func(tx *gorm.DB) error {...})`，函数内禁止起裸 goroutine 复用 `tx` |
| 指标 | Prometheus label 禁用 `uid` / `order_id`，SQL 指标用 `operation` + `table` 作 label |

---

## 12. 与团队字段词典的偏离及待确认事项

### 12.1 偏离登记（需规范 owner 登记豁免）

| 偏离项 | 团队词典原规则 | 本规范规则 | 原因 |
|---|---|---|---|
| `create_time` / `update_time` | 词典将其列为**禁用别名**，要求只用 `created_at` / `updated_at` | **必须**有，类型 `BIGINT UNSIGNED` 毫秒时间戳 | 业务需要毫秒精度与跨系统传递 |
| `created_at` / `updated_at` 类型 | `TIMESTAMPTZ(6)` 微秒精度 | MySQL `DATETIME(6)`（微秒精度，**不带时区**，依赖实例与会话为 UTC） | 本规范指定；无 2038 上限 |
| 审计表 `before` / `after` | 沿用 `before` / `after` | 改 `before_snapshot` / `after_snapshot` | `before` 为 MySQL 保留字 |
| 主键与业务 ID | 词典仍写 `BIGINT IDENTITY` 单主键 | `id`（`BIGINT UNSIGNED AUTO_INCREMENT`）仍为主键，另加 `oid`（`CHAR(36)` UUIDv7）作业务 ID；逻辑外键用 `{实体}_oid` 而不是 `{实体}_id` | 业务不依赖自增 `id`，对外与跨服务只用 `oid`，见 [id-generation.md](id-generation.md) |

### 12.2 待确认

| # | 问题 | 当前处理 |
|---|---|---|
| 1 | 业务**为什么**要 `create_time` + `created_at` 两套时间？若仅为兼容历史系统，建议长期收敛为一套 | 已按要求保留两套，并在 §4.1 划分职责 |
| 2 | `created_at` / `updated_at` 已定为 `DATETIME(6)`（无 2038 上限）；代价是不带时区，须保证全链路 UTC | 已在 §2.2 / §4.1 写明 |
| 3 | “业务不允许依赖主键 ID”的口径 | 本文理解为：保留自增 `id` 作库内主键，业务一律用应用层生成的 `oid`；`id` 不出表、不进 API / Kafka / Redis、不被其他表引用（§4.3） |
| 4 | 单表预警线 2000 万行 / 20 GB 是经验值 | 需 DBA 按硬件校准 |
| 5 | **验证范围**：已在 MySQL 8.4.11 实测——§4.10 建表模板（`id` 自增 + `oid` + `uk_*_oid`）；1062 报错信息里带索引名（`orders.uk_orders_oid` 与业务唯一键可区分）；`utf8mb4_0900_bin` 下大写 `oid` 查不到小写行、且**能绕过唯一索引写入重复行**（所以 `oid` 必须小写归一）；失败 INSERT 会消耗自增号（空洞）；按 `oid` 查询走 `uk_orders_oid`、乐观锁更新（`WHERE oid=? AND version=?`）与游标分页；§5.1 分区（只有 `PRIMARY KEY(id)` 时报 1503；`(id, create_time)` + `uk_oid(oid, create_time)` 可建，同一 `oid` 落在不同分区可重复；`REORGANIZE PARTITION`）。更早已实测：在线 DDL（INSTANT / INPLACE / INVISIBLE）、`INET6_ATON`、`clientFoundRows`、`DECIMAL(38,18)` 往返、CHECK 约束。`DATETIME(6)`（`DEFAULT` / `ON UPDATE CURRENT_TIMESTAMP(6)`：插入默认值、真实变更才刷新、同值 UPDATE 不刷新、精度不一致报 1294 / 1067、会话时区 +08:00 写入本地时间、可写 `9999-12-31`）。**未验证**：ProxySQL、gh-ost、半同步复制、性能数字（P99 / TPS 阈值）、2000 万行 / 20 GB 预警线 | 上述未验证项落地前须在目标环境验证 |
| 6 | Axon 现有 `idgen` 只有 UUIDv4，需改为 UUIDv7（`NewV7`） | 属代码改动，未处理 |
| 7 | Axon `CLAUDE.md` 写“ID 生成：Snowflake”，与本规范（`id` 自增 + `oid` UUIDv7，禁 Snowflake）冲突 | 以本规范为准，`CLAUDE.md` 未改 |

---

## 版本记录

| 版本 | 日期 | 摘要 |
|---|---|---|
| 1.1.1-draft | 2026-10-22 | 删除 `created_by` / `updated_by` / `deleted_at`；取消软删除与 `is_alive` 生成列，允许物理 DELETE（§4.4） |
| 2.2.0-draft | 2026-10-24 | `created_at` / `updated_at` 由 `TIMESTAMP` 改为 `DATETIME(6) DEFAULT CURRENT_TIMESTAMP(6)`（`updated_at` 另加 `ON UPDATE CURRENT_TIMESTAMP(6)`）；全库禁用 `TIMESTAMP`；写明不带时区带来的会话时区约束 |
| 2.1.0-draft | 2026-10-22 | **ID 方案调整**：保留自增 `id`（`BIGINT UNSIGNED`）为库内主键，新增 `oid`（`CHAR(36)` UUIDv7）为业务 ID，二者并存；逻辑外键改 `{实体}_oid`；分区表主键 `(id, create_time)`；`readTimeout` 35s。取代 2.0.0 的“UUIDv7 `BINARY(16)` 作主键、禁止自增” |
| 2.0.0-draft | 2026-10-22 | 作为本团队 MySQL 8.4 主规范；时间字段改为 `create_time`/`update_time`（毫秒）+ `created_at`/`updated_at`（`TIMESTAMP`）；明确 UUIDv7 主键与 `BINARY(16)` 选型；分区键改 `create_time` |
| 1.0.0-draft | 2026-10-22 | 初稿 |
