# 11 执行引擎

## 1. 角色

执行引擎承担"**把一张已存好的 Graph 实际跑起来**"这件事。它是一个独立进程 —— `ExecutionManager`（[`backend/executor/manager.py`](../backend/executor/manager.py)，约 1900 行），不直接面向 HTTP，所有任务都从 RabbitMQ 的 `execution_run_queue` / `execution_cancel_queue` 取。

```
[AgentServer / Scheduler / webhook ingress] ──► RabbitMQ execution exchange
                                                            │
                                                            ▼
                                                ┌──────────────────────┐
                                                │  ExecutionManager    │
                                                │  ├─ ThreadPoolExecutor (N workers)
                                                │  ├─ ExecutionProcessor (per-thread)
                                                │  ├─ ClusterLock (Redis)
                                                │  └─ event_bus.publish (Redis Sharded PubSub)
                                                └──────────────────────┘
                                                            │
                                                            ▼
                                                  Postgres (execution.py)
                                                  Redis (events for WS)
```

## 2. 进程结构

### 2.1 `ExecutionManager`

- 继承 `AppProcess`；
- 启动时：
  1. 连接 Redis / RabbitMQ；
  2. 拉起 `SyncRabbitMQ` consumer，订阅两个队列（`auto_ack=False`，手动 ack）；
  3. 创建 `ThreadPoolExecutor(max_workers=settings.num_graph_workers)`；
  4. 对每条消息调 `_handle_run_message()` 提交到线程池。
- 收到 SIGTERM：
  - 取消 RabbitMQ 消费；
  - 等待当前 graph 跑完（带 timeout）；
  - 未完成的执行：消息自然回队（manual ack 没发出）。

### 2.2 `ExecutionProcessor`

每个 worker 线程持有一个独立的 `ExecutionProcessor`（thread-local，避免共享状态）。它负责整张图的遍历与节点调度：

```
ExecutionProcessor
  ├─ on_graph_execution(entry)            # 入口
  ├─ on_node_execution(node_input)        # 单节点
  ├─ register_next_executions(...)        # 沿 link 推下游
  ├─ resolve_block_cost / charge_*        # 计费
  └─ event_bus.publish(...)               # 事件分发
```

## 3. 一次 Graph 执行的端到端时序

```
1.  AgentServer.execute_graph()
       └─ create_graph_execution(status=QUEUED)
       └─ rabbitmq.publish(execution_run_queue, GraphExecutionEntry)

2.  ExecutionManager._consume_execution_run()
       └─ ThreadPoolExecutor.submit(_handle_run_message, msg)

3.  worker thread
       ├─ ExecutionProcessor.on_graph_execution(entry):
       │     ├─ get_graph(id, version)          # data/graph.py
       │     ├─ update_status(RUNNING)          # data/execution.py
       │     ├─ collect 起始 nodes (in_degree == 0)
       │     ├─ asyncio.gather(*[on_node_execution(n) for n in start])
       │     │     │
       │     │     └─ for each node concurrently:
       │     │           ├─ validate_input()
       │     │           ├─ creds_manager.acquire(user_id, cred_id)
       │     │           ├─ check_balance() / resolve_block_cost()
       │     │           ├─ block.execute(input_data, **extra_kwargs)
       │     │           │     └─ async generator yield (out_pin, value)
       │     │           ├─ upsert_execution_input/output(...)
       │     │           ├─ stats = NodeExecutionStats(walltime, items, provider_cost)
       │     │           ├─ charge_reconciled_usage(stats)
       │     │           ├─ update_status(COMPLETED|FAILED|REVIEW)
       │     │           ├─ event_bus.publish(NODE_EXEC_UPDATE)
       │     │           └─ register_next_executions(node, outputs)
       │     │                  └─ 静态 link 即刻入队；动态 link 等齐再入队
       │     │
       │     ├─ wait until queue empty
       │     ├─ aggregate stats (cost / duration / token usage)
       │     ├─ update_status(COMPLETED|FAILED)
       │     └─ event_bus.publish(GRAPH_EXEC_UPDATE)
       │
       └─ rabbitmq.basic_ack(msg)
```

## 4. 节点执行的细节

### 4.1 输入装配

一个节点的输入来自 4 个源（按优先级合并）：

1. `AgentNode.constantInput`（用户在 Builder 里硬编码）；
2. 上游节点的输出（按 `AgentNodeLink` 引用，sink_name → source_name）；
3. `AgentGraphExecution.inputs`（启动时整张图的入参，绑定到入口节点的 pin）；
4. `nodesInputMasks`（单次执行的覆盖掩码，例如重试时调高 LLM temperature）。

执行前通过 Pydantic schema 校验；不通过则节点 `FAILED`，整图按设置决定继续或终止。

### 4.2 凭证注入

凭证字段命名固定为 `credentials` 或 `*_credentials`，类型 `CredentialsMetaInput[ProviderName, FieldType]`。执行器：

```python
for field_name, cred_meta in node_block.get_credentials_fields():
    cred = await creds_manager.acquire(user_id=user_id, credentials_id=cred_meta.id)
    extra_exec_kwargs[field_name] = cred
```

`creds_manager.acquire()` 内部：
- 查 Redis `lock:cred_refresh:{cred_id}`（防多次刷新）；
- 必要时调对应 OAuth handler `refresh_token()`；
- 加另一把 `acquire lock` 防止 in-flight token 被并发刷新替换。

### 4.3 静态 vs 动态 Link

- **动态**（`isStatic=false`）：默认。下游节点必须等齐所有 input pin 才入队。一次执行只触发一次。
- **静态**（`isStatic=true`）：源 pin 的输出可被多次复用。常见于：常量、配置、被循环 Block 反复消费的迭代项。

### 4.4 嵌套执行

`AgentExecutorBlock`（[`backend/blocks/agent.py`](../backend/blocks/agent.py)）能在节点内部触发**子 graph** 执行，写入 `parentGraphExecutionId`。父 graph 的状态会聚合子 graph 的成本。

### 4.5 HITL（Human in the Loop）

- `HumanReviewBlock`：节点执行到一半 yield 一个 `PendingHumanReview`，状态置 `REVIEW`，**返回**而不阻塞 worker；
- 用户在前端批准/拒绝/编辑 → `POST /api/executions/action`；
- 服务端写回 `human_review.update_review_status()`，并发布事件让 ExecutionManager 重新拉起后续；
- 还支持 `auto_approve_{graphExecId}_{nodeId}` 缓存，跳过等待。

### 4.6 取消

`POST /api/executions/{exec_id}/cancel`：
- 写入 `AgentGraphExecution.status=CANCELLED`；
- 发 RabbitMQ 到 `execution_cancel_queue`；
- ExecutionManager 收到后 set 一个 cancel flag；
- 当前节点的 block 通常基于 asyncio cancellation 优雅退出（block 自己的协程要 `try/except CancelledError`）；
- 下游 link 不再 register。

## 5. 计费（与 [16](./16-credit-billing.md) 配合）

### 5.1 BlockCostType 枚举

| 类型       | 触发                                  | 说明                                            |
| ---------- | ------------------------------------- | ----------------------------------------------- |
| `RUN`      | 执行前预扣                            | 固定单次成本                                    |
| `BYTE`     | 执行前预扣（按输入字节）              | 用于上传/解析体量正比的计算                     |
| `SECOND`   | 执行后结算（按 walltime）             | `cost_divisor` 决定粒度                         |
| `ITEMS`    | 执行后结算（按返回项数）              | 例：搜索 N 条                                   |
| `COST_USD` | 执行后结算                            | 使用 provider 实际成本（如 Replicate 计算回报） |
| `TOKENS`   | 执行后结算                            | LLM 按 token 计价（查表）                       |

### 5.2 流程

1. **预检**：`resolve_block_cost(node, input)` 估算 RUN/BYTE 成本，调 `credit.spend_credits()`；不足 → 抛 `InsufficientBalanceError` → 节点 `FAILED`。
2. **执行**：收集 `NodeExecutionStats`（walltime / item_count / provider_cost / token usage）。
3. **对账**：`charge_reconciled_usage()` 算实际成本，差额补扣或返还。
4. **平台成本日志**：写 `PlatformCostLog`（自家烧的钱，不等于用户额度）。

### 5.3 低余额保护

- `balance <= 0` 时拒绝任何带成本的 Block；
- 触发 `LowBalanceData` / `ZeroBalanceData` 通知（节流，按 Redis 缓存窗口）；
- Discord 告警走 `backend/util/metrics.py`。

## 6. 集群锁

`backend/executor/cluster_lock.py`：

- 基于 Redis SETNX + TTL 的可重入分布式锁；
- 用于：
  - 同一 graph 同一时刻只有一个执行实例（防用户重复点击）；
  - 凭证刷新；
  - 某些 Block 内的并发限制。

## 7. 状态生成器

`backend/executor/activity_status_generator.py`：

- 把节点执行进度浓缩成"已完成 X / 总 N，预估剩余 T"；
- 写入 `GraphExecution.stats`，用于前端进度条。

## 8. 自动审批 / Auto Mod

`backend/executor/automod/`：

- 拦截敏感操作 Block（`is_sensitive_action=true`）；
- 走规则引擎自动批 / 拒；
- 不通过的回落到 `PendingHumanReview`。

## 9. 错误与可观测性

- `try/except` 包住 `block.execute()`，错误写入 `NodeExecution.executionData.error`；
- Sentry 自动上报；
- Prometheus：
  - `autogpt_graph_executions_total{status}`
  - `autogpt_block_executions_total{block_type,status}`
  - `autogpt_block_duration_seconds{block_type}`（Histogram）
- `backend/monitoring/late_execution_monitor.py` 周期扫超时执行并告警。

## 10. 测试

- `backend/executor/manager_test.py` —— 集成测试，对真实 Postgres / RabbitMQ；
- `backend/executor/manager_*_test.py` —— 计费、低余额、auto credentials、insufficient funds 等专项；
- `backend/executor/cluster_lock_test.py`、`backend/executor/scheduler_test.py`、`backend/executor/simulator_test.py`。

## 11. 容器与扩容

- 容器：`docker-compose.platform.yml: executor`；
- 扩容：水平复制 executor 容器，RabbitMQ Quorum Queue 天然分发；
- 单容器 worker 数：`num_graph_workers`（默认 10）；
- CPU/IO 重的 Block（如 ffmpeg / pandas）建议拆出独立 worker pool（未实现专用，靠总数控制）。
