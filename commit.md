---
title: "Commit Message 规范"
version: "1.0.0"
last_modified: "2026-04-26"
source: "规范版本库0.0.2 / commit-message.md"
---

# Commit Message 规范

# Commit Message 规范

> 版本：V1.0.0 | 状态：生效 | 适用范围：所有 Git 仓库


---

## 1. 总则

采用 **Conventional Commits** 规范，所有提交信息必须通过 CI 中的 `commitlint` 检查。不合规的 Commit 将被自动拒绝。


---

## 2. 格式

```
<type>(<scope>): <subject>

[可选 body]

[可选 footer]
```

### 2.1 Type（必填）

| Type | 说明  | 示例  |
|------|-----|-----|
| feat | 新功能 | feat(trading): add limit order support |
| fix  | Bug 修复 | fix(wallet): correct balance calculation |
| docs | 文档变更 | docs(api): update authentication guide |
| style | 代码格式（不影响逻辑） | style: fix linting errors |
| refactor | 重构（非新功能非修复） | refactor(order): extract validation logic |
| perf | 性能优化 | perf(matching): optimize order book lookup |
| test | 测试相关 | test(wallet): add withdrawal unit tests |
| build | 构建/依赖变更 | build: upgrade Go to 1.22 |
| ci   | CI 配置变更 | ci: add security scanning stage |
| chore | 其他杂项 | chore: update .gitignore |
| revert | 回滚  | revert: feat(trading): add limit order |

### 2.2 Scope（推荐填写）

Scope 对应业务模块，团队约定以下标准 Scope：

`trading`, `wallet`, `account`, `risk`, `matching`, `gateway`, `common`, `infra`, `ci`, `docs`

### 2.3 Subject（必填）

* 全小写，英文
* 不超过 72 个字符
* 使用祈使语气：`add` 而非 `added` / `adds`
* 不以句号结尾

### 2.4 Body（大变更必填）

* 解释 **为什么** 做这个变更，而非 **做了什么**
* 每行不超过 100 字符
* 与 subject 间空一行

### 2.5 Footer

* 关联 Issue：`Closes #123` 或 `Refs #456`
* 破坏性变更：`BREAKING CHANGE: 描述`


---

## 3. 示例

### 普通提交

```
feat(wallet): add multi-chain deposit address generation

Support generating deposit addresses for ETH, BSC, and Polygon
networks. Each address is derived from the user's master key using
BIP-44 path derivation.

Closes #789
```

### 破坏性变更

```
feat(api)!: change order response format to v2 schema

BREAKING CHANGE: The order response now uses nested objects for
price and quantity fields. Clients using v1 format must migrate
to /api/v2/orders endpoint.
```

### Bug 修复

```
fix(trading): prevent duplicate order submission on network retry

Add idempotency key check in order creation handler to prevent
duplicate orders when client retries on timeout.

Closes #456
```


---

## 4. CI 自动检测

### 4.1 commitlint 配置

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'build', 'ci', 'chore', 'revert'
    ]],
    'scope-enum': [1, 'always', [
      'trading', 'wallet', 'account', 'risk',
      'matching', 'gateway', 'common', 'infra', 'ci', 'docs'
    ]],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [2, 'always', 100],
  },
};
```

### 4.2 Git Hook（本地）

使用 `husky` + `commitlint` 在本地 commit 时校验：

```bash
# .husky/commit-msg
npx --no -- commitlint --edit "$1"
```

### 4.3 CI Pipeline

GitLab CI 中增加检查阶段：

```yaml
commit-lint:
  stage: validate
  script:
    - npx commitlint --from $CI_MERGE_REQUEST_DIFF_BASE_SHA --to HEAD
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```


---

## 5. 禁止事项

* ❌ 中文提交信息
* ❌ 无意义信息：`update`, `fix bug`, `wip`, `test`
* ❌ 超大提交：单次 Commit 修改超过 500 行（应拆分）
* ❌ 混合变更：一个 Commit 包含不相关的多个修改