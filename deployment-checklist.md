---
title: "部署清单"
version: "1.0.0"
last_modified: "2026-04-26"
source: "规范版本库0.0.2 / 部署清单.md"
---

# 部署清单

# 部署清单

## 1. 发布基本信息

* 项目 / 系统：核心交易系统
* 发布版本：v1.8.0
* 发布日期：2026-05-12
* 变更窗口：2026-05-12 23:00 – 2026-05-13 01:00（UTC+8）
* 发布负责人：aaron（工号 E1024）
* 发布等级：🟡 重要架构重构

## 2. 发布审批记录

| 角色  | 审批人 | 审批时间 | 结论  |
|-----|-----|------|-----|
| Tech Lead | chrs | 2026-05-10 14:20 | 通过  |
| SRE | liwei | 2026-05-10 16:00 | 通过  |
| DBA | zhangyu | 2026-05-11 10:30 | 通过  |
| 业务负责人 | aaron | 2026-05-11 11:00 | 通过  |

## 3. 迭代需求与任务

| 任务 ID | 需求描述 | 研发  | 需求来源 | 备注  |
|-------|------|-----|------|-----|
| Jira-101 | 支付链路优化 | chrs | aaron | 解决信用卡超时问题 |
| Jira-105 | 订单幂等性加固 | linda | product | 防重复下单 |

## 4. 代码仓库与构建

| 服务  | Git 仓库 | 发布分支 | Commit ID |
|-----|--------|------|-----------|
| order-api | [git@github.com](mailto:git@github.com):org/order.git | release/v1.8.0 | a1b2c3d4  |
| pay-service | [git@github.com](mailto:git@github.com):org/pay.git | release/v1.8.0 | e5f6g7h8  |

## 5. 镜像与回滚准备

| 服务  | 本次发布镜像 Tag | 稳定版回滚 Tag | 部署集群 |
|-----|------------|-----------|------|
| order-api | [registry.com/order:v1.8.0_build10](http://registry.com/order:v1.8.0_build10) | [registry.com/order:v1.7.3_build22](http://registry.com/order:v1.7.3_build22) | EKS-Cluster-01 |
| pay-service | [registry.com/pay:v1.8.0_build12](http://registry.com/pay:v1.8.0_build12) | [registry.com/pay:v1.7.3_build25](http://registry.com/pay:v1.7.3_build25) | EKS-Cluster-01 |

## 6. 配置变动

* 配置中心（Apollo）：
  * `order.service.timeout`：2000 → 5000
  * `feature.flag.new-pay-gate`：false → true
* 环境变量（K8s）：
  * 新增 `MAX_RETRY_ATTEMPTS=3`

## 7. 数据库变动

* 目标库：order_db_master
* 变更类型：DDL
* 预估耗时：约 3 分钟
* 是否 Forward-only：否（支持回滚）

### 7.1 SQL 审核记录

| 审核人 | 审核时间 | 审核结论 | 关联工单 |
|-----|------|------|------|
| zhangyu（DBA） | 2026-05-11 10:30 | 通过   | DBA-2026-0512-01 |

### 7.2 执行脚本（DDL / DML）

```sql
-- migrations/000234_add_payment_unique_idx.up.sql
ALTER TABLE t_payment_record
  ADD CONSTRAINT uk_serial_no UNIQUE (serial_no);
```

### 7.3 回滚脚本

```sql
-- migrations/000234_add_payment_unique_idx.down.sql
ALTER TABLE t_payment_record
  DROP CONSTRAINT uk_serial_no;
```

## 8. 中间件与基础设施变动

* Redis：新增 Key 前缀 `cache:order:v2:`，TTL 30min
* Kafka：新增 Topic `prod_order_completed`，分区 12、副本 3、保留期 7d
* 对象存储：更新 bucket `exchange-prod-order` 访问策略，开放 ServiceA 读权限

## 9. 发布前检查

- [x] 测试报告签字（QA：linda，2026-05-10）
- [x] P0 用例全部通过（12 / 12）
- [x] 安全扫描通过（SonarQube 无 Blocker）
- [x] 性能压测通过（订单 TPS 3000，P99 180ms）
- [x] 监控告警已配置
- [x] 灰度方案已评审
- [x] On-call 已到岗（aaron / chrs / liwei）
- [x] 上下游已通知（wallet / risk 团队）

## 10. 发布步骤与操作清单

> 按顺序执行，⏸ Checkpoint 需人工确认再继续。


1. 执行数据库 Migration（§7.2）→ 校验索引创建成功
2. order-api 灰度 10% → ⏸ 观察 10 分钟（错误率 < 0.5%、P99 < 300ms）
3. order-api 扩量 50% → ⏸ 观察 10 分钟
4. order-api 全量发布
5. pay-service 灰度 10% → ⏸ 观察 10 分钟
6. pay-service 全量发布
7. 打开开关 `feature.flag.new-pay-gate=true`
8. ⏸ 全量观察 30 分钟，无异常宣布发布成功

## 11. 监控与告警

* 核心 Dashboard：<https://grafana.internal/d/trade-core>
* 关键指标阈值：
  * 接口错误率 < 1%
  * P99 延迟 < 500ms
  * 订单创建成功率 > 99.5%
* 日志关键词告警：`panic` / `FATAL` / `timeout` / `deadlock`
* 告警通道：企业微信"交易系统发布群"、值班手机 13800000000

## 12. 发布后验证

- [ ] 冒烟用例跑通（下单 / 支付 / 查询 / 撤单）
- [ ] 核心指标连续观察 30 分钟无异常
- [ ] 业务对账一致（订单数、金额与 wallet 流水）
- [ ] 客服反馈通道 1 小时内无异常工单

## 13. 应急回滚预案

* **触发条件**：
  * 接口错误率 > 5% 持续 3 分钟
  * 订单创建成功率 < 98%
  * P99 延迟 > 1s 持续 5 分钟
* **RTO 目标**：≤ 10 分钟
* **回滚步骤**：

  
  1. 发布负责人 aaron 下达回滚指令，同步发布群
  2. 应用回滚：镜像 Tag 回退（参考 §5 稳定版 Tag）
  3. 配置回滚：Apollo 回滚至版本 `v1.7.3-config`
  4. 数据库回滚：执行 §7.3 回滚脚本（Forward-only 变更执行补偿方案）
  5. 跑冒烟用例 + 核心指标复核
* **Forward-only 补偿方案**：本次无 Forward-only 变更；如涉及，典型做法——通过新增字段覆盖旧字段、数据修正脚本补齐
* **通知路径**：aaron（决策） → 发布群 → 客服主管 → wallet / risk 团队