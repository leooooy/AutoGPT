# 16 积分与计费

## 1. 概念模型

平台用 **Credit（积分）** 作为执行 Block 的"燃料"。Credit 与真实美元的换算：

```
1 USD = 1,000,000 microdollars = 100 credits（默认换算）
```

实际全程以 **microdollars (int)** 存储，避免浮点精度问题；UI 上展示时再换算。

| 数据                                                | 用途                                                  |
| --------------------------------------------------- | ----------------------------------------------------- |
| `User.subscriptionTier`                             | 订阅档位 (NO_TIER / BASIC / PRO / MAX / BUSINESS / ENTERPRISE) |
| `User.topUpConfig`                                  | 自动充值阈值 / 金额（JSON）                           |
| `UserBalance.balance`                               | 当前余额（microdollars，缓存快照）                    |
| `CreditTransaction`                                 | 所有进出账流水（含正负 amount）                       |
| `CreditRefundRequest`                               | 退款申请                                              |
| `PlatformCostLog`                                   | **平台对外侧成本**（自家烧的钱），用于成本分析        |

## 2. 余额与流水

### 2.1 数据一致性

- **`UserBalance` 是流水表 `CreditTransaction.amount` 的累加**；
- 写入流水与更新余额**必须同事务**（[`backend/data/credit.py`](../backend/data/credit.py)）；
- `CreditTransaction.runningBalance` 冗余存当时余额，便于审计和漂移检测。

### 2.2 transactionKey 幂等

`CreditTransaction` 的复合主键是 `(transactionKey, userId)`。`transactionKey` 是业务侧生成的唯一键（UUID / Stripe event id / `graph_exec_id:node_exec_id` 等）。同一 key 重复提交会因主键冲突被拒，不会重复扣费。

### 2.3 流水类型

| `CreditTransactionType` | 触发场景                                  | amount 符号       |
| ----------------------- | ----------------------------------------- | ----------------- |
| `TOP_UP`                | Stripe 充值成功                           | +                 |
| `USAGE`                 | Block 执行扣费                            | -                 |
| `GRANT`                 | 管理员赠送 / 注册赠送 / 推广              | +                 |
| `REFUND`                | 客服退款（关联 `CreditRefundRequest`）    | +                 |
| `CARD_CHECK`            | Stripe 验卡（occasional 1¢ 测试）         | 0 / 微负后回滚    |
| `SUBSCRIPTION`          | 订阅 cycle 内的赠送 credit                | +                 |

### 2.4 metadata

`CreditTransaction.metadata` JSON 存 context：

```jsonc
// USAGE
{
  "transactionKey": "graph_exec/uuid:node_exec/uuid",
  "graph_exec_id": "...",
  "node_exec_id": "...",
  "block": "AnthropicLLMBlock",
  "input": "...truncated...",
  "reason": "tokens",
  "tokens": { "input": 1234, "output": 567 },
  "platform_cost_microdollars": 4500
}
```

可在 admin 侧根据 metadata 复盘账单。

## 3. 扣费时机与方式

详见 [11 执行引擎 §5](./11-execution-engine.md#5-计费与-16-配合)。一句话：**预扣 + 对账**。

```
[执行 Block 前]
   resolve_block_cost(node, input)         # 估 RUN/BYTE 成本
   spend_credits(...)                      # 真扣
   if InsufficientBalance: 节点 FAILED, 整图按设置决定继续

[执行 Block 中]
   收集 NodeExecutionStats:
     walltime_ms, item_count, llm_tokens, provider_cost_usd

[执行 Block 后]
   actual_cost = compute(stats, costs[])
   delta = actual_cost - reserved_cost
   if delta > 0:  spend_credits(delta)     # 补扣
   if delta < 0:  grant_credits(-delta, type=REFUND, reason=reconcile)
   log_platform_cost(stats, provider_cost) # 写 PlatformCostLog
```

`BlockCostType` 枚举决定算法：`RUN / BYTE / SECOND / ITEMS / COST_USD / TOKENS`。

## 4. 充值与订阅

### 4.1 Stripe 充值

```
1. POST /api/credits/top-up        → 创建 Stripe Checkout session，返回 url
2. 用户在 Stripe 完成支付
3. Stripe → POST /api/credits/stripe_webhook  (公开端点，按签名校验)
4. 服务端：
     - 校验 event.type == "checkout.session.completed"
     - transactionKey = stripe_event_id (强幂等)
     - grant_credits(amount, type=TOP_UP, metadata={stripe_event_id, ...})
```

### 4.2 自动充值（Auto Top-Up）

`User.topUpConfig` 字段：

```jsonc
{
  "enabled": true,
  "threshold_microdollars": 100000,    // <= $0.10 时
  "amount_microdollars": 1000000       // 充 $1.00
}
```

`spend_credits()` 完成后若新余额 ≤ 阈值，触发后台 Stripe charge（用户保存的支付方式）。失败则下发邮件 + 关闭 auto top-up（防扣破产卡）。

### 4.3 订阅

- Stripe 订阅 → `CreditTransaction` 类型 `SUBSCRIPTION` 周期赠送 credit；
- 同时更新 `User.subscriptionTier`（影响 Copilot 限流倍率）；
- 订阅取消 / 试用到期由 webhook + Scheduler 一次性任务处理。

## 5. 平台成本日志（`PlatformCostLog`）

> 平台为某次执行真正烧给第三方多少钱（OpenAI / Anthropic / Replicate 调用费），与用户被扣的 credit 不必相等（平台可以加价 / 补贴）。

写入：

```python
log_platform_cost(
    user_id=...,
    graph_exec_id=...,
    node_exec_id=...,
    provider="openai",                     # 标准化小写
    cost_microdollars=4500,
    input_tokens=1234, output_tokens=567,
    cache_read_tokens=100, cache_creation_tokens=20,
    tracking_type="llm_tokens",
    tracking_amount=1801,
)
```

用途：
- 成本分析仪表盘（按 provider / 按 user / 时间窗口）；
- 检测异常（某个 user 突然烧超量 → 可能滥用）；
- 决定 BlockCost 表是否要调价。

## 6. 退款

```
1. POST /api/credits/refund (body: { transactionKey, reason })
   → CreditRefundRequest (status=PENDING)

2. Admin 在后台审批 → status=APPROVED
3. 服务端：
   - grant_credits(original_amount, type=REFUND, transactionKey="refund:<orig_key>")
   - 通知用户
4. 拒绝 → status=REJECTED + 通知
```

退款流水的 `transactionKey` 用 `refund:<orig_key>` 形式，确保不与原扣费冲突，且二次提交退款幂等。

## 7. 通知触发

`spend_credits()` 后会判断：

- 余额 ≤ 0：发布 `ZeroBalanceData` → NotificationManager → 邮件 + Web Push（如订阅）；
- 余额 ≤ `LOW_BALANCE_THRESHOLD`：`LowBalanceData`，节流（窗口 24h，避免轰炸）；
- 自动充值失败：单独事件 + Sentry。

通知通道与模板见 [18 通知系统](./18-notifications.md)。

## 8. CLI / Admin 工具

```bash
# 给某用户赠送 credit
poetry run cli credits grant --user-id ... --amount-credits 100 --reason "promo"

# 查询某次执行的扣费明细
SELECT amount, runningBalance, metadata
FROM platform."CreditTransaction"
WHERE metadata->>'graph_exec_id' = '...'
ORDER BY "createdAt";

# 平台成本聚合
SELECT provider, SUM("costMicrodollars")/1.0e6 AS usd, COUNT(*)
FROM platform."PlatformCostLog"
WHERE "createdAt" > now() - interval '1 day'
GROUP BY provider ORDER BY usd DESC;
```

## 9. 关键代码

| 文件                                                           | 内容                                            |
| -------------------------------------------------------------- | ----------------------------------------------- |
| [`backend/data/credit.py`](../backend/data/credit.py)          | `UserCreditBase` / `spend_credits` / `grant_credits` / 流水查询 |
| [`backend/data/platform_cost.py`](../backend/data/platform_cost.py) | 平台成本写入 / 聚合                       |
| [`backend/data/block_cost_config.py`](../backend/data/block_cost_config.py) | 每个 Block 的成本规则                  |
| [`backend/executor/cost_tracking.py`](../backend/executor/cost_tracking.py) | 执行过程中的成本计算 / 对账             |
| [`backend/executor/billing.py`](../backend/executor/billing.py) | Stripe webhook 处理、订阅周期 / reconciliation |
| [`backend/executor/auto_credentials.py`](../backend/executor/auto_credentials.py) | 自动充值                            |

## 10. 测试

围绕 credit 的测试非常全面（`backend/data/credit_*_test.py`）：

- `credit_test.py` —— 基础读写 / runningBalance；
- `credit_concurrency_test.py` —— 并发扣费 race condition；
- `credit_underflow_test.py` —— 防止负余额；
- `credit_ceiling_test.py` —— 上限约束；
- `credit_refund_test.py` —— 退款幂等；
- `credit_subscription_test.py` —— 订阅周期赠送；
- `credit_metadata_test.py`、`credit_user_balance_migration_test.py`、`credit_integration_test.py`；
- 执行侧：`backend/executor/manager_low_balance_test.py`、`manager_insufficient_funds_test.py`、`manager_cost_tracking_test.py`、`billing_reconciliation_test.py`。

## 11. 易错点

| 坑                                                       | 正确做法                                             |
| -------------------------------------------------------- | ---------------------------------------------------- |
| 在事务外算余额然后扣（TOCTOU）                            | 在 SQL 事务内 `SELECT ... FOR UPDATE` 或用 increment |
| 直接用 float 存 USD                                       | 用 microdollars (int)                                |
| 重复消费同一 Stripe webhook                               | 用 `stripe_event_id` 当 `transactionKey`             |
| LLM Block 异常但已花了 token                              | 仍要写 PlatformCostLog；用户侧按规则扣或免扣          |
| 给用户充值后没失效缓存                                    | `get_user_by_id` LRU 要 invalidate                   |
| 退款时给原 `transactionKey` 取反                          | 永远用 `refund:<orig>` 防主键冲突                     |
