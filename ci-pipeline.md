---
title: "CI Pipeline 标准化规范"
version: "1.0.0"
last_modified: "2026-04-26"
source: "规范版本库0.0.2 / ci-pipeline.md"
---

# CI Pipeline 标准化规范

# CI Pipeline 标准化规范

> 版本：V1.0.0 | 状态：生效 | 适用范围：所有 GitLab CI Pipeline


---

## 1. 总则

所有服务的 CI Pipeline 必须遵循统一的阶段定义和质量门禁。Pipeline 配置使用共享模板（CI Include），各服务按需扩展。


---

## 2. Pipeline 阶段定义

```
┌──────────┬────────────┬──────────┬──────────┬──────────┬──────────┐
│ validate │   build    │   test   │   scan   │  package │  deploy  │
└──────────┴────────────┴──────────┴──────────┴──────────┴──────────┘
```

| 阶段  | 内容  | 失败影响 |
|-----|-----|------|
| validate | Lint、Commit 格式、命名规范检查 | 阻塞 Pipeline |
| build | 编译构建 | 阻塞 Pipeline |
| test | 单元测试 + 覆盖率检查 | 阻塞 Pipeline |
| scan | 安全扫描（SAST + 依赖漏洞 + 容器扫描） | 阻塞 Pipeline（Critical/High） |
| package | Docker 构建 + 推送 Harbor | 阻塞 Pipeline |
| deploy | 自动部署到 Dev/Staging（Prod 需手动审批） | 不阻塞  |


---

## 3. 共享模板（CI Include）

### 3.1 模板结构

```
ci-templates/
├── go.yml              # Go 服务通用模板
├── react.yml           # React 前端通用模板
├── flutter.yml         # Flutter 移动端通用模板
├── security.yml        # 安全扫描模板
├── docker.yml          # 镜像构建模板
└── deploy.yml          # 部署模板
```

### 3.2 服务引用方式

```yaml
# 各服务 .gitlab-ci.yml
include:
  - project: 'infra/ci-templates'
    ref: main
    file:
      - '/go.yml'
      - '/security.yml'
      - '/docker.yml'
      - '/deploy.yml'

variables:
  SERVICE_NAME: order-service
  GO_VERSION: "1.22"
  COVERAGE_THRESHOLD: "80"
```


---

## 4. Go 服务 Pipeline 模板

```yaml
# ci-templates/go.yml
.go-validate:
  stage: validate
  image: golangci/golangci-lint:latest
  script:
    - golangci-lint run --config .golangci.yml ./...
    - npx commitlint --from $CI_MERGE_REQUEST_DIFF_BASE_SHA --to HEAD
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

.go-build:
  stage: build
  image: golang:$GO_VERSION
  script:
    - go mod download
    - go build -o bin/$SERVICE_NAME ./cmd/server/
  artifacts:
    paths: [bin/]
    expire_in: 1h

.go-test:
  stage: test
  image: golang:$GO_VERSION
  script:
    - go test -race -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out
    - |
      COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
      if (( $(echo "$COVERAGE < $COVERAGE_THRESHOLD" | bc -l) )); then
        echo "Coverage $COVERAGE% below threshold $COVERAGE_THRESHOLD%"
        exit 1
      fi
  coverage: '/total:.*\s(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
```


---

## 5. React 前端 Pipeline 模板

```yaml
# ci-templates/react.yml
.react-validate:
  stage: validate
  image: node:20
  script:
    - npm ci
    - npx eslint --max-warnings 0 src/
    - npx tsc --noEmit

.react-test:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm run test -- --coverage --watchAll=false
    - |
      COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
      if (( $(echo "$COVERAGE < 70" | bc -l) )); then exit 1; fi
```


---

## 6. 质量门禁

### 6.1 MR 合并条件（全部必须通过）

* ✅ Pipeline 全绿
* ✅ 覆盖率达标（Go ≥ 80%，React ≥ 70%，Flutter ≥ 70%）
* ✅ 零 Critical/High 安全漏洞
* ✅ Code Review Approved
* ✅ Commit Message 格式合规

### 6.2 主干保护

* main/master 分支设为 Protected
* 禁止直接 Push，必须通过 MR
* 合并需满足上述所有门禁


---

## 7. Pipeline 性能要求

| 指标  | 目标  |
|-----|-----|
| MR Pipeline 总耗时 | ≤ 10 分钟 |
| 主干 Pipeline 总耗时 | ≤ 15 分钟 |
| 构建缓存命中率 | ≥ 80% |

优化手段：

* Go: 缓存 `$GOPATH/pkg/mod`
* Node: 缓存 `node_modules`
* Docker: 多阶段构建 + BuildKit 缓存
* 并行执行独立 Job