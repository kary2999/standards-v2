# Team Standards · 公开规范镜像

> Go 微服务团队规范的**只读公开镜像**，供 Team Standards App 自动同步使用。
> 本仓库内容由规范 owner 维护，**禁止 PR 改动**——改规范请在私有源仓库提，CI 同步过来。

---

## 用途

Team Standards macOS App 通过 GitLab archive zip 拉取本仓库 `*.md` 文件，
覆盖 `~/.claude/skills/go-team-standards/references/` 下的规范引用，
让团队所有人无需重装 App 即可拿到最新规范。

## 文件清单

| 文件 | 内容 |
|---|---|
| `database.md` | 数据库设计与变更规范（PG 18 / 命名 / 分区 / Migration / 安全 / Kafka / Redis） |
| `mysql-database.md` | MySQL 数据库设计与变更规范（MySQL 8.x / InnoDB；UUIDv7 主键 / 时间字段 / 分区 / Migration / 安全） |
| `field-naming.md` | 全局统一字段命名规范（v1.0.1，9 大类 + 禁用别名 + 缩写白名单） |
| `naming-logging.md` | 日志命名规范（snake_case / trace_id / 字段约定） |
| `naming-visual.png` | 命名规范可视化速查图 |
| `api-design.md` | 接口设计原则（路径 / 错误码 / 分页 / 版本） |
| `api-doc-example.md` | 接口文档完整范例（67 KB 实战参照） |
| `error-codes.md` | 错误码定义与 errno 包用法 |
| `go-style.md` | Go 代码风格（命名 / 包组织 / 错误处理 / 并发） |
| `tech-design-example.md` | 技术方案 7 段范例（50 KB 实战参照） |
| `testing.md` | 测试规范（单元 / 集成 / e2e） |
| `code-review.md` | Code Review 规范（评审清单 / 流程） |
| `ci-pipeline.md` | CI/CD 流水线规范 |
| `commit.md` | Git commit 规范（Conventional Commits） |
| `deployment-checklist.md` | 部署上线 checklist |
| `feature-flags.md` | 特性开关 / 灰度发布规范 |
| `cursor-usage.md` | Cursor 使用约定 |
| `glossary.md` | 团队术语表 |
| `meeting-minutes.md` | 会议纪要模板 |
| `tixuebj-template-simple.md` | 提测文档模板（手填版，OrangeCat skill 自动版替代品） |

## 版本

每个 .md 文件头部 frontmatter 标注 `version` + `last_modified`，
仓库整体版本以 `latest commit SHA` 标识。

## 同步频率

App 启动时静默检查（fail 不阻断），用户也可手动点「检查规范更新」按钮。

## 反馈

规范本身有问题：联系团队规范 owner（IM / Issue），不要直接改本仓库。
