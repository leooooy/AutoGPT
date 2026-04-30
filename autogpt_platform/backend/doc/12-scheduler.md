# 12 调度器（Scheduler）

## 1. 角色

`Scheduler` 是一个独立进程，负责所有"按时触发"的事情：

- **用户定义的 Cron 任务**：由前端在 Library / Builder 中创建，触发某个 Graph 的执行；
- **平台内部周期任务**：监控、清理、汇总通知等；
- **一次性任务**：例如订阅试用到期、HITL 超时回收。

源码：[`backend/executor/scheduler.py`](../backend/executor/scheduler.py)，进程类 `Scheduler(AppService)`。

## 2. 框架

底层是 [APScheduler](https://apscheduler.readthedocs.io/) 的 `BackgroundScheduler`：

- **Trigger**：`CronTrigger`（标准 cron 表达式 + 时区）、`DateTrigger`（一次性）。
- **Job Store**：`SQLAlchemyJobStore`，落到 PostgreSQL（与业务库同库不同 schema）。重启不丢任务。
- **Executor**：`ThreadPoolExecutor`，并发执行回调。

## 3. 数据模型

业务侧的"用户定义任务"持久化在 [`AgentExecutionScheduledJob`](./05-data-model.md) 表，关键字段：

| 字段                          | 含义                                            |
| ----------------------------- | ----------------------------------------------- |
| `id`                          | UUID                                            |
| `userId`                      | 拥有者                                          |
| `agentGraphId / version`      | 要执行的 graph                                  |
| `cron`                        | cron 表达式（用户时区）                         |
| `inputs` / `credentialInputs` | 启动输入                                        |
| `nextRunAt`                   | 下一次执行时间（冗余，便于查询）                |
| `isActive`                    | 软关闭                                          |
| `name` / `metadata`           | UI 用                                           |

启动 Scheduler 进程时，会从这张表把所有 `isActive=true` 的任务 mirror 到 APScheduler 的 jobstore，并维护增删改的同步。

## 4. RPC 接口

`Scheduler(AppService)` 通过 Pyro5 暴露：

| 方法                                        | 调用方                | 含义                                  |
| ------------------------------------------- | --------------------- | ------------------------------------- |
| `add_execution_schedule(...)`               | AgentServer / preset  | 新建用户定时任务                      |
| `update_execution_schedule(id, ...)`        | AgentServer           | 改 cron / inputs / 状态               |
| `remove_execution_schedule(id)`             | AgentServer           | 删除                                  |
| `list_execution_schedules(user_id)`         | AgentServer           | 列表                                  |
| `trigger_one_off_job(...)`                  | 内部                  | 加一次性任务                          |

用户改完调度后，AgentServer RPC 调 Scheduler，Scheduler 同步落到 DB + 内存中的 APScheduler。

## 5. 触发到执行的链路

```
APScheduler 触发 → Scheduler 进程内回调
   └─ 反查 AgentExecutionScheduledJob 拿最新 inputs/credentials
   └─ 调 AgentServer execute_graph()（本质是发 RabbitMQ）
        └─ ExecutionManager 消费并执行（参见 [11](./11-execution-engine.md)）
```

调度器不直接执行 Block —— 仅"触发"，执行交给 ExecutionManager，避免长任务把 scheduler 阻塞。

## 6. 内置周期任务

除了用户任务外，平台还有若干"内部 Job"，由 Scheduler 在启动时注册：

| Job                                | 周期         | 文件                                                         |
| ---------------------------------- | ------------ | ------------------------------------------------------------ |
| `accuracy_monitor`                 | 每小时       | `backend/monitoring/accuracy_monitor.py`                     |
| `block_error_monitor`              | 每 5 分钟    | `backend/monitoring/block_error_monitor.py`                  |
| `late_execution_monitor`           | 每分钟       | `backend/monitoring/late_execution_monitor.py`               |
| `notification_monitor`             | 每 10 分钟   | `backend/monitoring/notification_monitor.py`                 |
| 周期通知聚合 / 摘要                | 每天 / 每周  | `backend/notifications/notifications.py`                     |
| Copilot session 清理                | 每 6 小时    | `backend/copilot/session_cleanup.py`                          |
| 凭证刷新（提前预热）                | 每 30 分钟   | `backend/integrations/creds_manager.py` 内的提前刷新逻辑      |

> 周期值以代码为准，可能调整。

## 7. Listener 与告警

- `job_listener` —— 任务执行完写日志；失败上报 Sentry；
- `job_missed_listener` —— 错过执行时间（如 Scheduler 停机）写 Discord 告警；
- `job_max_instances_listener` —— 同一任务并发实例超限告警；
- 所有告警都 throttle，避免炸群。

## 8. 时区

- 用户任务 cron 按 `User.timezone` 解析；
- 平台内部任务统一 UTC；
- APScheduler 内部存 UTC，触发时按 `tz` 转换。

## 9. 实现要点

### 9.1 Mirror 同步

新增/改/删任务时，AgentServer 先写 DB，再 RPC 通知 Scheduler 同步内存中的 APScheduler。如果 Scheduler 暂时不可达：

- DB 已经写入了真值；
- Scheduler 重启会从 DB 全量加载；
- 用户调度不会真正"丢"，但可能错过下次执行 —— 由 `job_missed_listener` 报警。

### 9.2 Cron 校验

`croniter` 校验表达式合法性（在 RPC 入口做），避免脏数据进 jobstore。

### 9.3 一次性任务

`DateTrigger(run_date)` 跑完自动从 jobstore 删除。例如：

- 订阅试用到期；
- HITL 超过 24h 未审 → 自动 reject；
- Webhook 临时绑定失效。

## 10. 部署

```bash
# 单进程
poetry run scheduler
```

`docker-compose.platform.yml: scheduler_server`：
- 端口 8003（RPC）；
- 依赖 db / redis / rabbitmq / database_manager 启动完成。

> 生产建议**单实例运行 Scheduler**：APScheduler 默认不是分布式调度器；多副本会让同一个 cron 任务被多次触发。如果将来要 HA，需要做 leader election（Redis 锁）或换调度框架。

## 11. 测试

- `backend/executor/scheduler_test.py` —— RPC 接口与 cron 解析；
- 内部任务的逻辑测试在各自模块的 `*_test.py`（如 `late_execution_monitor` 自带）。
