---
# 测试
title: "命名规范"
version: "1.1.0"
last_modified: "2026-04-29"
source: "技术规范.2026.04.29 / 命名规范.md"
---

# 命名规范

# 命名与日志规范手册 (2026)

## 一、 核心意义 (The Philosophy)

* **极致一致性：** 确保新人与 AI 都能读懂代码，消除沟通成本。
* **运维透明化：** 支撑业务全链路追踪，确保每一分钱的资源投入都能在监控中精准定位。


------

## 二、 命名规范 (Naming Conventions)

### 1. 项目与仓库 (Repository)

* **命名格式：** `[业务域]-[子系统]-[功能/模块]-[类型]`
  * *示例：* `finance-payment-gateway-service`
* **后缀区分：**
  * `-service`: 微服务
  * `-lib`: 公共库 (Library)
  * `-job`: 定时任务/批处理
  * `-web`: 前端 Web 端
  * `-app`: 移动端 (Flutter/Native)

### 2. Go 语言规范 (Golang Specific)

* **文件命名：** 全小写 + 下划线（snake_case）。测试文件必须以 `_test.go` 结尾。
* **包名 (Package)：** 简短且具语义化。使用单数形式。
  * *正确：* `transport`, `util`, `model`
  * *错误：* `transport_layer`, `utils`, `models`
* **接口 (Interface)：** 惯用 `-er` 后缀。如 `Reader`, `Writer`, `CustomerManager`。
* **变量/结构体：**
  * **导出 (Public)：** PascalCase
  * **非导出 (Private)：** camelCase
  * **缩写处理：** 必须保持全大写或全小写。如 `JSONProcessor` (√) 而非 `JsonProcessor` (×)。

### 3. 中间件命名 (Middleware) - 由IDP控制

* **消息队列 (Topic)：** `[环境]_[服务名]_[业务语义]_[动作]`
  * *项目内：* `prod_order_payStatus_updated`
  * *跨项目：* `env_A_2_B_biz` (如 `prod_pay_to_order_record`)
* **Redis Key：** `[业务名]:[模块]:[唯一标识]`
  * *注：* 冒号用于 Redis 客户端自动分组。


------

## 三、 链路追踪规范 (Traceability)

**遵循 W3C Trace Context 标准：**

| **字段** | **定义** | **格式** | 生成  | **示例** |
|-----|-----|-----|-----|-----|
| **trace_id** | 标识整个请求链路 | 32位十六进制字符串 | Otel | `4bf92f3577b34da6a3ce929d0e0e4736` |
| **span_id** | 标识具体操作单位 | 16位十六进制字符串 | Otel | `00f067aa0ba902b7` |
| **parent_span_id** | 调用链来源标记 | 16位十六进制字符串 | Otel | `542ee1d33f733671` |


------

## 四、 日志打印规范 (Logging Standards)

### 1. 基础格式

* **必须使用 JSON 格式**，确保机器可读与日志系统兼容。
* **日志级别定义：** `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`。

### 2. 字段命名约定 (snake_case)

#### 核心基础字段 (Base Fields)

`env`, `time`, `level`, `msg`, `trace_id`, `service`, `caller`, `cost_ms`

#### 请求日志 (Request Logs)

用于记录 HTTP/gRPC 调用上下文：

`action`, `status`, `uid`, `uri`, `ip`, `req`, `res`

#### 业务日志 (Business Logs)

用于记录核心业务逻辑变动：

`event`, `entity`, `ext_info`, `ref_id`


------

## 五、 治理门禁 (The Enforcement)

> **注意：** > 1. 所有命名规范已植入 GitLab CI 提交检测。
>
> 
> 2. 不符合规范的代码将无法通过静态扫描（golangci-lint），禁止提交 PR。