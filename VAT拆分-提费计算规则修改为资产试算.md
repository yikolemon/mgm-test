# fix(VAT拆分): 提费计算规则修改为资产试算 — 改动说明

## 改动概述

Commit `8728ed6a` 将 VAT 拆分场景下"不同意税费拆分"的提费（利率提升）计算规则，从 app-server 本地反推改为走下游资产试算口径。核心变更在 `tryCalculateV3` 中引入双试算并发编排，替代原有的 `calculateFakeAmount` 本地推导。

## 背景

历史技术方案（`doc/VAT税费拆分技术方案.md`）中定义的提费逻辑：

- **申请阶段（apply）**：`interestRate = interestRate / fakeDiscountRate`
- **试算阶段（tryCalculateV3）**：app-server 在本地用 `calculateFakeAmount(amount, fakeDiscountRate)` 按比例反推优惠前金额

此方案的隐患：本地用固定比例反推叠加下游 client 尾插，导致前台展示的优惠前金额、假折扣金额与最终借款申请口径存在偏差。

## 旧逻辑（改动前）

### tryCalculateV3 链路

```
couponService.calculateV3(realLoanReq)
  └── 返回 SimulateCouponDto（含真实优惠券试算结果）
        └── fillInRepayScheduleV3(realCalculate, feeTable)
              ├── 真实优惠后字段 ← 真实试算结果
              ├── 优惠前字段 ← calculateFakeAmount(金额, fakeDiscountRate)  # 本地反推
              ├── 假折扣金额 ← 本地按比例推导
              └── discountAmount ← 本地汇总
```

`calculateFakeAmount` 实现：

```java
private BigDecimal calculateFakeAmount(BigDecimal amount, BigDecimal rate) {
    return amount.divide(rate, 2, RoundingMode.UP);
}
```

### apply 链路

```java
interestRate = interestRate.divide(fakeDiscountRate, 2, RoundingMode.UP);
```

## 新逻辑（改动后）

### tryCalculateV3 链路

```
并发执行：
  ├── 真实优惠券试算：couponService.calculateV3(realLoanReq)
  └── 假折扣试算：listingFeatureClient.calculateV1(fakeLoanReq)
        └── fakeLoanReq.interestRate = getIncreasedInterestRate(原始利率)
              └── fillInRepayScheduleV3(realCalculate, fakeCalculate, feeTable)
                    ├── 真实优惠后字段 ← 真实试算结果
                    ├── 优惠前字段 ← 假折扣试算结果（original* 系列）
                    ├── fakeDiscountAmount = fake.总还款金额 - real.总还款金额
                    ├── couponDiscountAmount = real.totalDiscountAmount
                    └── discountAmount = fakeDiscountAmount + couponDiscountAmount
```

### 关键方法提取

原 `apply` 中的提费计算被提取为共享方法：

```java
private BigDecimal getIncreasedInterestRate(BigDecimal interestRate) {
    return interestRate.divide(fakeDiscountRate, 2, RoundingMode.UP);
}
```

`apply` 和 `tryCalculateV3` 的假折扣请求共用此方法，确保试算与申请口径一致。

### 假折扣请求构造

```java
private LoanCalculateV1Request buildFakeCalculateReq(TryCalculateReq request,
                                                     ProductConfigDto prodConfig,
                                                     PataFeeItem pataFeeItem) {
    LoanCalculateV1Request fakeLoanReq = buildCalculateReq(request, prodConfig, pataFeeItem);
    fakeLoanReq.setInterestRate(getIncreasedInterestRate(fakeLoanReq.getInterestRate()));
    return fakeLoanReq;
}
```

### 并发编排

使用 JDK 21 虚拟线程 + `CompletableFuture`：

```java
private final ExecutorService executor = Executors.newThreadPerTaskExecutor(
        VirtualThreadFactoryUtil.virtualThreadFactoryWithName("loanService"));

// 在 tryCalculateV3 中：
CompletableFuture<SimulateCouponDto> realFuture = CompletableFuture.supplyAsync(
        SupplierWrapper.of(() -> couponService.calculateV3(realLoanReq)), executor);
CompletableFuture<LoanCalculateVO> fakeFuture = CompletableFuture.supplyAsync(
        SupplierWrapper.of(() -> calculateFakeDiscount(fakeLoanReq)), executor);
joinFutures(realFuture, fakeFuture);
```

`joinFutures` 负责异常传播：假折扣试算失败时抛 `SYSTEM_ERROR`，不吞异常、不返回局部结果。

### fillInRepayScheduleV3 重构

| 字段 | 旧来源 | 新来源 |
|------|--------|--------|
| `interest` | 真实试算 | 真实试算（不变） |
| `originalInterest` | `calculateFakeAmount(i.interest, rate)` | `fakeItem.interest` |
| `originalServiceFee` | `calculateFakeAmount(i.serviceFee, rate)` | `fakeItem.serviceFee` |
| `originalTaxFee` | `calculateFakeAmount(i.taxFee, rate)` | `fakeItem.taxFee` |
| `originalRepayAmount`（顶层） | 真实试算 + 本地假折扣求和 | `fake.totalRepaymentAmount` |
| `originalRepayAmount`（明细） | `principal + debtOriginalLoanFee` | `fakeItem.repaymentAmount` |
| `originalLoanFee` | 明细求和 | `sum(fakeItem 各项费用)` |
| `fakeDiscountAmount` | 按比例计算的本地推导值 | `fake.totalRepaymentAmount - real.totalRepaymentAmount` |
| `loanFee`（明细） | **Bug**: `interest + serviceFeeDiscount + taxFee` | `interest + serviceFee + taxFee`（修正 `serviceFeeDiscount` → `serviceFeeAfterDiscount`） |

### 删除的方法

`calculateFakeAmount(BigDecimal, BigDecimal)` — 所有调用点被替换为直接从假折扣试算结果读取。

## 测试变更

1. **重构**：提取 `prepareTryCalculateV3BaseMocks()`、`buildTryCalculateReq()`、`buildSimulateCouponDto()` 等辅助方法，消除大量重复 mock 代码
2. **新增验证**：
   - `testTryCalculateV3_WithFakeDiscount_ShouldUseDownstreamFakeCalculate`：验证假折扣试算结果正确用于 `original*` 字段，`fakeDiscountAmount = fake.总金额 - real.总金额`
   - `testTryCalculateV3_FakeCalculateFailed_ShouldThrowBizException`：假折扣试算失败时抛出 `BizException`
   - `testTryCalculateV3_CouponCalculateFailed_ShouldKeepOriginalException`：真实试算失败时保持原有异常语义
3. **删除了** 2 个旧测试：`testTryCalculateV3_WithFakeDiscount`（覆盖不完整）、`testTryCalculateV3_WithCustomerPropertyOther`（语义重复）
4. **参数捕获验证**：使用 `ArgumentCaptor` 验证 `fakeLoanReq.interestRate` 等于 `getIncreasedInterestRate(0.05)` = `0.10`

## 影响范围

| 维度 | 说明 |
|------|------|
| 接口 | `/loan/tryCalculateV3` — 响应字段来源变化，结构不变 |
| | `/loan/apply` — 行为不变，仅抽提取消方法为 `getIncreasedInterestRate` |
| 外部依赖 | 新增一次 `listingFeatureClient.calculateV1` 调用（假折扣适用时） |
| 性能 | 假折扣试算通过虚拟线程与真实试算并发，不串行放大耗时 |
| 配置 | 不变，复用既有 `fake.discount.rate` |

## 与历史方案的关系

| 历史方案（VAT税费拆分技术方案.md） | 本次 fix | 一致性 |
|--------------------------------|----------|--------|
| 不同意拆分时提费：`interestRate / fakeDiscountRate` | 提取为 `getIncreasedInterestRate`，apply 行为不变 | 一致 |
| 试算返回 `originalRepayAmount` 用于展示 | 不再本地反推，改为下游试算提供 | **优化**：消除尾差偏差 |
| 不同意拆分时不提供假折扣 | 假折扣改为走下游试算口径 | 行为不变，来源变更 |
