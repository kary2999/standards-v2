---
title: "特性开关与分支管理办法"
version: "1.1.0"
last_modified: "2026-05-14"
source: "技术规范.2026.05.14 / 特性开关与分支管理办法.md"
---

# 特性开关与分支管理办法

## 1. 设计哲学：部署与发布解耦

我们通过特性开关将**部署 (Deployment)** 与**发布 (Release)** 彻底分离：

- **部署**：随时进行。代码合入主干并部署到 EKS，但功能默认关闭。
- **发布**：业务决策。在 IDP 平台上点击开关，瞬间对指定用户开启功能，无需重新跑 CI/CD。

---

## 2. 特性开关 (Feature Flags) 管控体系

### 2.1 注册与定义 (IDP 侧)

开发者在 IDP 平台（如 Backstage）的"特性开关"模块进行注册：

- **Key**：全局唯一标识（如 `enable-new-payment-gateway`）。
- **Type**：开关类型（Boolean、Multivariate 权重、Targeting 针对性开启）。
- **Ownership**：关联所属微服务与负责人。
- **Persistence**：配置持久化在 GitOps 仓的 ConfigMap 或专用的 Flag 服务中。

### 2.2 Go 代码集成规范

在 `go-common` 中集成统一的 FF 客户端，**禁止自行实现**。

```go
// 标准调用示例
if ff.IsEnabled(ctx, "enable-new-payment-gateway", userID) {
    // 执行新需求逻辑 (需求 A)
    return newPaymentService.Process(ctx)
}
// 默认执行旧逻辑
return oldPaymentService.Process(ctx)
```

### 2.3 开关的生命周期管理

- **默认状态**：所有新 Feature Flag 默认值必须为 `OFF`。
- **清理机制**：需求上线 100% 稳定后，流程中心会定期发起任务，提醒开发者删除代码中的 `if` 判断并下线开关，防止"逻辑碎屑"堆积。

---

## 3. GitOps

针对 5 个需求并行的挑战，我们将 PR 流程打造为自动化实验室。

### 3.1 分支管理办法

详细分支流转见 `feature-flags-branch.png`（与本文同目录）。

### 3.2 自动化预览环境（The "5-to-4" Solution）

当 5 个需求并行时，每一个 PR 都会触发以下动作：

1. **动态 Namespace**：基于 `pr-<id>-<service>` 创建独立空间。
2. **配置克隆**：自动克隆生产环境的基准配置，但仅替换当前 PR 涉及的镜像。
3. **流量切分**：利用 **Istio VirtualService**，通过特殊的 Header（如 `x-env: pr-123`）将流量引导至该预览版本。
4. **成本控制**：预览环境仅保留 48 小时，过期或 PR 关闭后由 `Janitor` 脚本自动回收，节省 EKS 资源费。

---

## 4. 协同作战：当需求发生变更时

### 场景 A：5 个需求要撤掉 1 个（5 变 4）

- **PR 层面**：直接 Close 掉那个撤销需求的 PR。由于它是基于 GitOps 独立分支和临时 Namespace 的，关闭动作会触发资源清理，**主干分支完全不受影响**。
- **分支层面**：直接删掉 release 重新从 main 拉取，合并想要上线的 4 个。
- **代码层面**：即使部分代码已合入主干，由于其受 **Feature Flag** 保护且默认值为 `OFF`，该功能在生产环境依然不可见，确保了系统的整体稳定性。

### 场景 B：紧急线上回滚

- **操作**：无需重新构建镜像。
- **动作**：在 IDP 界面将对应的 Feature Flag 瞬间置为 `OFF`。
- **时效**：秒级生效。这比 Git 归档和全量同步（分钟级）快得多，极大地降低了故障平均恢复时间（MTTR）。

---

## 5. 提交格式标准

采用增强型的 **Conventional Commits** 风格，强制关联需求 ID。

**格式**：`<type>: [reqID] <subject> (@user)`

**示例**：

```
feat: [req101] 增加钱包提现最大额度限制 (@max)
fix:  [req203] 修复成交记录分页超时 (@lina)
docs: [req101] 补充 FF 配置说明 (@max)
```

### Type 类型表

| Type | 含义 |
|---|---|
| `feat` | 新功能（需求内） |
| `fix` | 修补 Bug |
| `docs` | 文档变更 |
| `style` | 格式变动（不影响逻辑） |
| `refactor` | 重构（既不修复 Bug 也不加功能） |
| `perf` | 性能优化 |
| `test` | 增加测试 |
| `chore` | 构建过程或辅助工具的变动 |
| `revert` | 回滚某个之前的 Commit |

---

## 6. 治理与审计

### 6.1 权限控制（The Gates）

- **开发环境**：开发者可自由开关。
- **生产环境**：只有架构师或运维负责人有权操作开关变更，且所有变更同步推送至 Slack 频道并记录在审计日志中。

### 6.2 性能预警

- 由于 Feature Flag 涉及配置拉取，`go-common` 客户端需具备**本地缓存（Local Cache）**能力，确保不会因为频繁请求配置中心而增加 API 延迟。

---

## 7. 总结清单

| 模块 | 核心任务 | 交付物 |
|---|---|---|
| **Feature Flag** | 封装 SDK 并建立 IDP 管理界面 | 业务发布与部署彻底解耦 |
| **PR Workflow** | 自动化预览环境搭建与回收 | 5 个并行需求的隔离验证环境 |
| **GitOps** | 跨仓自动同步镜像版本 | 清晰的配置变更链路与快速回滚能力 |
