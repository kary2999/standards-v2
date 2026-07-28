---
title: "Code Review 规范"
version: "1.0.0"
last_modified: "2026-04-26"
source: "规范版本库0.0.2 / code-review.md"
---

# Code Review 规范

# Code Review 规范

> 版本：V1.0.0 | 状态：生效 | 适用范围：所有代码仓库


---

## 1. 总则

Code Review 是代码质量的最后一道防线。所有代码变更必须经过至少一位合格评审人 Approve 后方可合并，核心模块需要两位。


---

## 2. MR/PR 提交规范

### 2.1 MR 标题

格式与 Commit Message 一致：`<type>(<scope>): <subject>`

### 2.2 MR 描述模板

```markdown
## 变更说明
<!-- 简述本次变更的目的和内容 -->

## 变更类型
- [ ] feat: 新功能
- [ ] fix: Bug 修复
- [ ] refactor: 重构
- [ ] perf: 性能优化
- [ ] docs: 文档
- [ ] test: 测试
- [ ] chore: 其他

## 影响范围
<!-- 列出受影响的模块/服务 -->

## 测试情况
- [ ] 单元测试已通过
- [ ] 集成测试已通过
- [ ] 手动测试已验证

## 关联 Issue
Closes #xxx

## 截图/日志（如适用）
<!-- 附上 UI 变更截图或关键日志 -->

## Checklist
- [ ] 代码符合编码规范
- [ ] 无硬编码的密钥/密码
- [ ] 已更新相关文档
- [ ] 数据库变更已提供 Migration
- [ ] 已考虑向后兼容性
```

### 2.3 MR 大小限制

* 理想：**200 行以内**
* 上限：**500 行**（超出需拆分或提前与 Reviewer 沟通）
* 大重构：可申请例外，但需附详细说明


---

## 3. 评审人指派

### 3.1 指派规则

| 变更类型 | 最少 Approve 数 | 评审人要求 |
|------|--------------|-------|
| 普通业务代码 | 1            | 同组开发者 |
| 核心模块（交易/钱包/风控） | 2            | 含 Tech Lead 或模块 Owner |
| 架构变更/新框架引入 | 2            | Tech Lead + 架构师 |
| 数据库 Schema 变更 | 2            | Tech Lead + DBA |
| CI/CD 配置变更 | 1            | DevOps 或 Tech Lead |
| 文档/配置 | 1            | 任意团队成员 |

### 3.2 CODEOWNERS

在仓库根目录维护 `CODEOWNERS` 文件，自动指派评审人：

```
# .gitlab/CODEOWNERS
*                       @backend-team
internal/trading/       @trading-team @tech-lead
internal/wallet/        @wallet-team @tech-lead
internal/risk/          @risk-team @tech-lead
migrations/             @tech-lead @dba
deployments/            @devops-team
.gitlab-ci.yml          @devops-team
```


---

## 4. 评审标准

### 4.1 必须检查项（阻塞合并）

* **安全性**：是否存在注入、硬编码密钥、敏感信息泄露
* **正确性**：逻辑是否正确，边界条件是否处理
* **错误处理**：error 是否被正确处理而非忽略
* **并发安全**：共享状态是否有竞态条件
* **性能**：是否有 N+1 查询、内存泄漏、不必要的大对象

### 4.2 建议检查项（不阻塞但应讨论）

* 命名是否清晰表达意图
* 是否有更简洁的实现方式
* 测试覆盖是否充分
* 是否需要补充文档

### 4.3 评审标记规范

| 前缀  | 含义  | 是否阻塞 |
|-----|-----|------|
| `[MUST]` | 必须修改 | 是    |
| `[NIT]` | 建议优化，不强制 | 否    |
| `[Q]` | 疑问，需要解释 | 否    |
| `[IDEA]` | 灵感，可以后续考虑 | 否    |


---

## 5. 评审 SLA

| 指标  | 要求  |
|-----|-----|
| 首次响应时间 | ≤ 4 小时（工作时间内） |
| 评审完成时间 | ≤ 1 个工作日 |
| 作者修复后再审 | ≤ 4 小时 |
| 紧急修复（P0 Hotfix） | ≤ 1 小时 |


---

## 6. 合并规则

* 合并方式：**Squash Merge**（保持主干历史整洁）
* 合并前必须满足：CI 全部通过 + 所有评审人 Approve + 无未解决 Thread
* 禁止自己 Approve 自己的 MR
* 合并后自动删除源分支
* 合并至 main/master 的 MR 禁止 Force Push


---

## 7. 特殊场景

### 7.1 Hotfix 流程

* P0/P1 故障可走加急评审通道
* 最少 1 位 Tech Lead Approve 即可合并
* 事后 24 小时内补充完整 Review

### 7.2 自动化检查前置

MR 创建时 CI 自动运行以下检查（结果展示在 MR 页面）：

* Lint 检查（golangci-lint / ESLint / dart analyze）
* 单元测试 + 覆盖率
* 安全扫描
* Commit Message 格式校验
* 构建验证