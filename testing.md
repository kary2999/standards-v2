---
title: "单元测试规范"
version: "1.0.0"
last_modified: "2026-04-26"
source: "团队原始（无更新）"
---

# 单元测试规范

# 单元测试规范

> 版本：V1.0.0 | 状态：生效 | 适用范围：所有代码仓库


---

## 1. 总则

单元测试是质量保障的基石。所有新增/修改的业务逻辑必须有对应的单元测试，覆盖率不达标的 MR 将被 CI 自动阻塞。


---

## 2. 覆盖率要求

| 技术栈 | 新代码覆盖率 | 整体覆盖率目标 | 核心模块覆盖率 |
|-----|--------|---------|---------|
| Go  | ≥ 80%  | ≥ 70%   | ≥ 90%   |
| React (TS) | ≥ 70%  | ≥ 60%   | ≥ 85%   |
| Flutter (Dart) | ≥ 70%  | ≥ 60%   | ≥ 85%   |

**核心模块定义：** 交易撮合、钱包转账、风控引擎、资金结算


---

## 3. Go 单测规范

### 3.1 测试框架

* 标准库 `testing` + `testify/assert`
* Mock：`go.uber.org/mock` (mockgen)
* HTTP Mock：`httptest`

### 3.2 表驱动测试（强制）

```go
func TestCalculateFee(t *testing.T) {
    tests := []struct {
        name     string
        amount   decimal.Decimal
        rate     decimal.Decimal
        expected decimal.Decimal
        wantErr  bool
    }{
        {
            name:     "normal fee calculation",
            amount:   decimal.NewFromFloat(1000),
            rate:     decimal.NewFromFloat(0.001),
            expected: decimal.NewFromFloat(1),
        },
        {
            name:    "negative amount rejected",
            amount:  decimal.NewFromFloat(-100),
            rate:    decimal.NewFromFloat(0.001),
            wantErr: true,
        },
        {
            name:     "zero amount returns zero fee",
            amount:   decimal.Zero,
            rate:     decimal.NewFromFloat(0.001),
            expected: decimal.Zero,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := CalculateFee(tt.amount, tt.rate)
            if tt.wantErr {
                assert.Error(t, err)
                return
            }
            assert.NoError(t, err)
            assert.True(t, tt.expected.Equal(got))
        })
    }
}
```

### 3.3 Mock 规范

* 只 Mock 外部依赖（DB、HTTP、消息队列），不 Mock 内部逻辑
* Mock 接口定义放在使用方，mockgen 自动生成
* 测试中验证 Mock 调用次数和参数


---

## 4. React 单测规范

### 4.1 测试框架

* Vitest / Jest + React Testing Library
* 禁止使用 Enzyme（已废弃）

### 4.2 测试原则

* 按用户行为测试，而非实现细节
* 使用 `screen.getByRole`, `getByText`，避免 `getByTestId`
* 异步操作使用 `waitFor` / `findBy`

```typescript
test('displays error when withdrawal amount exceeds balance', async () => {
  render(<WithdrawForm balance={1000} />);
  
  await userEvent.type(screen.getByRole('spinbutton', { name: /amount/i }), '1500');
  await userEvent.click(screen.getByRole('button', { name: /submit/i }));
  
  expect(screen.getByText(/insufficient balance/i)).toBeInTheDocument();
});
```


---

## 5. Flutter 单测规范

### 5.1 测试框架

* `flutter_test` + `mocktail`
* Widget 测试使用 `testWidgets`

### 5.2 测试分类

* **Unit Test**：纯业务逻辑（ViewModel、UseCase、Repository）
* **Widget Test**：组件渲染与交互
* **Golden Test**：UI 回归（关键页面）


---

## 6. 测试命名规范

统一格式：`Test{被测对象}_{场景}_{期望结果}`

```
TestOrderService_CreateOrder_Success
TestOrderService_CreateOrder_InsufficientBalance_ReturnsError
TestWithdrawForm_ExceedsBalance_ShowsError
```


---

## 7. 测试数据管理

* 使用 Factory 模式创建测试数据，禁止硬编码
* 测试间相互独立，不依赖执行顺序
* 测试完成后清理副作用（数据库记录、临时文件）
* 敏感数据（私钥、真实地址）不得出现在测试代码中


---

## 8. CI 集成

* 单元测试在每个 MR Pipeline 中自动运行
* 覆盖率报告自动展示在 MR 页面
* 覆盖率下降超过 2% 的 MR 自动标记警告
* 核心模块覆盖率低于阈值直接阻塞合并