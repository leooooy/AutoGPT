# 18 通知系统

## 1. 全景

```
[业务模块]
   │  notification_bus.publish(NotificationEventModel)
   ▼
[Redis Sharded PubSub]                      ←─ 给 WebSocket 推前端实时小红点
   │
   │  也可直接 rabbitmq.publish(immediate_notifications_v2)
   ▼
[RabbitMQ exchange `notifications`]
   ├─ immediate_notifications_v2     ─►  立即邮件 / Web Push
   ├─ summary_notifications_v2       ─►  每日 / 每周摘要
   ├─ batch_notifications_v2         ─►  批量聚合（同 user 同类型 1h 内合并）
   └─ failed_notifications_v2        ─►  失败重试（死信）
        │
        ▼
[NotificationManager 进程]
   ├─ Jinja2 渲染模板
   ├─ Postmark 发邮件
   ├─ pywebpush 发 Web Push
   └─ 失败 → 重投 failed_notifications_v2
```

源码：

- [`backend/notifications/`](../backend/notifications/) —— 进程实现 + 模板；
- [`backend/data/notifications.py`](../backend/data/notifications.py) —— 事件 schema；
- [`backend/data/notification_bus.py`](../backend/data/notification_bus.py) —— Redis 总线；
- [`backend/data/push_subscription.py`](../backend/data/push_subscription.py) / [`push_sender.py`](../backend/data/push_sender.py) —— Web Push。

## 2. 通知事件模型

[`backend/data/notifications.py`](../backend/data/notifications.py)：

```python
class NotificationType(str, Enum):
    AGENT_RUN
    ZERO_BALANCE
    LOW_BALANCE
    BLOCK_EXECUTION_FAILED
    CONTINUOUS_AGENT_ERROR
    DAILY_SUMMARY
    WEEKLY_SUMMARY
    MONTHLY_SUMMARY
    AGENT_APPROVED
    AGENT_REJECTED
    ...

class QueueType(str, Enum):
    IMMEDIATE   # 立即发
    BATCH       # 1h 聚合
    SUMMARY     # 每日 / 每周
    BACKOFF     # 指数退避
    ADMIN       # 管理员告警

class NotificationEventModel(BaseModel):
    user_id: str
    type: NotificationType
    data: AgentRunData | ZeroBalanceData | LowBalanceData | ...
    queue_type: QueueType
```

每种 `NotificationType` 都有对应的 `BaseNotificationData` 子类（typed payload），编译期校验。

## 3. NotificationManager 进程

[`backend/notifications/notifications.py`](../backend/notifications/notifications.py)

- 继承 `AppService`；
- 启动 4 个 RabbitMQ consumer，分别对应 4 个队列；
- Quorum queue + manual ack：失败可重投；
- 超过最大重试次数 → 入死信队列 + Sentry。

### 3.1 Immediate

最常见路径：业务模块 publish → 直接发邮件 + Push。例：
- Agent 执行完成；
- 余额为零；
- Auto top-up 失败。

### 3.2 Batch

对 `BLOCK_EXECUTION_FAILED` / `CONTINUOUS_AGENT_ERROR` 这类高频事件，按 `(user_id, type)` 聚合到 `UserNotificationBatch`：

- 每 user × 每 type 仅一行；
- 子事件写到 `NotificationEvent` 表；
- 1h 后由 Scheduler 触发"flush batch"，合并发一封邮件。

### 3.3 Summary

每天/每周/每月固定时间，扫所有用户 → 生成摘要邮件（开关在 `User.notifyOn{Daily,Weekly,Monthly}Summary`）。

### 3.4 Failed

- `failed_notifications_v2` 死信队列；
- 也是 retries 用：Backoff 策略由消息 header 控制；
- 持续失败的 PushSubscription 自动下线（`failCount >= 5`）。

## 4. 用户偏好

`User` 表的 `notify*` 字段开关每种通知：

```
notifyOnAgentRun, notifyOnZeroBalance, notifyOnLowBalance,
notifyOnBlockExecutionFailed, notifyOnContinuousAgentError,
notifyOnDailySummary, notifyOnWeeklySummary, notifyOnMonthlySummary,
notifyOnAgentApproved, notifyOnAgentRejected
```

`maxEmailsPerDay` 限制总条数（默认 3）。所有发件前都过滤一遍偏好。

## 5. 邮件

[`backend/notifications/email.py`](../backend/notifications/email.py)

- 客户端：`postmarker`；
- 模板：`backend/notifications/templates/`，Jinja2；
- 退订机制：邮件页脚带 `?token=...` 退订链接，token 是 HMAC（密钥 `UNSUBSCRIBE_SECRET_KEY`）；
- HTML + 纯文本双版本；
- 邮件 inline 资源走 cid。

## 6. Web Push

### 6.1 订阅

```
1. 前端请求 GET /api/push/vapid-key  → 公钥
2. 前端用 Service Worker subscribe(VAPID 公钥) 拿 endpoint
3. POST /api/push/subscribe { endpoint, p256dh, auth }
4. 后端写 PushSubscription 表
```

### 6.2 端点白名单

[`backend/data/push_subscription.py`](../backend/data/push_subscription.py)：

- 只允许：FCM (`fcm.googleapis.com`)、Mozilla Autopush、Apple Web Push；
- HTTPS 强制；
- 防 SSRF：拒绝任意 hostname。

### 6.3 发送

[`backend/data/push_sender.py`](../backend/data/push_sender.py)：

- `pywebpush` + VAPID 私钥；
- 失败计数：连续 5 次失败 → 删除订阅；
- 410 Gone → 立即删除（订阅失效）；
- VAPID claim email 来自 `VAPID_CLAIM_EMAIL`。

## 7. 关键场景

| 触发                                    | 类型                       | 通道           |
| --------------------------------------- | -------------------------- | -------------- |
| 用户启动的 graph 完成                   | `AGENT_RUN`                | Email + Push   |
| 余额为 0                                | `ZERO_BALANCE`             | Email + Push   |
| 余额低于阈值                            | `LOW_BALANCE`              | Email          |
| 节点反复失败                            | `CONTINUOUS_AGENT_ERROR`   | Email (Batch)  |
| Block 单次执行失败                      | `BLOCK_EXECUTION_FAILED`   | Email (Batch)  |
| Store 商品被审核通过 / 拒绝             | `AGENT_APPROVED / REJECTED`| Email          |
| 摘要                                    | `DAILY/WEEKLY/MONTHLY`     | Email          |
| Stripe webhook 异常                     | (Admin 类)                 | Discord 告警   |

## 8. 内部 Discord 告警

[`backend/util/metrics.py`](../backend/util/metrics.py) 提供 `discord_alert(...)`：

- 用于平台运营/系统告警（Stripe 失败、长执行、低余额风暴）；
- 通过 Discord webhook URL 发；
- 限流 + 缓存窗口防止刷屏。

## 9. 配置

| 环境变量                | 作用                                   |
| ----------------------- | -------------------------------------- |
| `POSTMARK_SERVER_TOKEN` | Postmark API                           |
| `EMAIL_FROM`            | 发件地址                               |
| `UNSUBSCRIBE_SECRET_KEY`| 退订 token HMAC                        |
| `VAPID_PRIVATE_KEY`     | Web Push                               |
| `VAPID_PUBLIC_KEY`      | Web Push                               |
| `VAPID_CLAIM_EMAIL`     | RFC 8292 claim email                   |
| `DISCORD_WEBHOOK_URL`   | 内部告警                               |

## 10. 测试

- `backend/notifications/test_notifications.py` —— 主流程；
- `backend/data/notifications_test.py` —— schema / queue 路由；
- `backend/data/push_subscription_test.py` —— 端点白名单；
- `backend/data/push_sender_test.py` —— pywebpush mock；
- `backend/data/notification_bus_test.py` —— Redis 总线；
- `backend/copilot/.../push_sender_test.py` —— 跟订阅相关的 Copilot 集成（按需）。

## 11. 写新通知类型的步骤

1. 在 `NotificationType` 枚举加新值；
2. 在 `data/notifications.py` 加对应 `BaseNotificationData` 子类（typed payload）；
3. 在 `notifications/templates/` 加 Jinja2 模板；
4. 在 `notifications/notifications.py` 的渲染器里 dispatch 到模板；
5. 业务侧：
   ```python
   notification_bus.publish(NotificationEventModel(
       user_id=user_id,
       type=NotificationType.MY_NEW_TYPE,
       data=MyNewTypeData(...),
       queue_type=QueueType.IMMEDIATE,
   ))
   ```
6. 加用户偏好开关字段（`notifyOnMyNewType`）+ schema migration；
7. 加测试。
