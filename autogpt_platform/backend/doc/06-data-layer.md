# 06 数据访问层（`backend/data/`）

## 1. 概览

`backend/data/` 是后端唯一直接接触 Prisma / Redis / RabbitMQ 的层。所有业务模块（API / executor / copilot / scheduler / notifications）只通过这里读写数据，从而：

- 把 ORM/缓存/队列的细节封装在一处；
- 让无 DB 直连的轻量进程通过 `DatabaseManager` 的 RPC 调用同样接口；
- 单测时只 mock 这层即可。

## 2. 模块清单（按职责）

### 2.1 连接 / 客户端

| 文件                          | 内容                                                                                            |
| ----------------------------- | ----------------------------------------------------------------------------------------------- |
| [`db.py`](../backend/data/db.py) | 全局 Prisma 实例 `prisma`、`transaction(timeout=30s)` 上下文管理器、连接池参数、`query_raw_with_schema()` |
| [`db_manager.py`](../backend/data/db_manager.py) | `DatabaseManager(AppService)`，对外 RPC 暴露 `graph_db()` / `execution_db()` / `library_db()` / `store_db()` / `chat_db()` / `search()` 等 |
| [`db_accessors.py`](../backend/data/db_accessors.py) | 在线/离线模式下的 import 路由：直连用 Prisma，远程用 RPC client |
| [`redis_client.py`](../backend/data/redis_client.py) | Redis Cluster 客户端工厂（同步 + 异步），处理 address remap / 健康检查 |
| [`redis_helpers.py`](../backend/data/redis_helpers.py) | 常用 helper：分布式锁、流操作 |
| [`rabbitmq.py`](../backend/data/rabbitmq.py) | Exchange/Queue 拓扑模型、`SyncRabbitMQ` / `AsyncRabbitMQ`；启动时 `declare_topology()` |
| [`event_bus.py`](../backend/data/event_bus.py) | 泛型 `BaseRedisEventBus[M]`，封装 Sharded PubSub；用于执行/通知事件 |
| [`notification_bus.py`](../backend/data/notification_bus.py) | 通知事件总线（基于 `event_bus.py`） |

### 2.2 业务实体 CRUD

| 文件                                    | 内容                                                                                |
| --------------------------------------- | ----------------------------------------------------------------------------------- |
| [`graph.py`](../backend/data/graph.py) | `GraphModel` / `NodeModel` / `Link`；`save_graph()` 事务保存；`get_graph()` 含完整节点树 |
| [`execution.py`](../backend/data/execution.py) | `GraphExecution` / `NodeExecution`；`create_graph_execution()`、`update_node_execution_status()`、`upsert_execution_input/output()` |
| [`block.py`](../backend/data/block.py) | `initialize_blocks()` 启动时同步 `AgentBlock` 元数据                                 |
| [`block_cost_config.py`](../backend/data/block_cost_config.py) | `BLOCK_COSTS` 字典：每种 Block 的成本规则                       |
| [`credit.py`](../backend/data/credit.py) | `UserCreditBase` 抽象类；`spend_credits()` / `grant_credits()` / `get_transaction_history()` |
| [`user.py`](../backend/data/user.py) | `get_or_create_user()`、`get_user_by_id/email()`（LRU 1000/300s）                    |
| [`integrations.py`](../backend/data/integrations.py) | Webhook 元数据 CRUD；`Webhook` / `WebhookWithRelations` 模型              |
| [`notifications.py`](../backend/data/notifications.py) | 通知事件 schema（AgentRunData / ZeroBalanceData / ...）；`QueueType`     |
| [`onboarding.py`](../backend/data/onboarding.py) | 引导步骤推进                                                                  |
| [`human_review.py`](../backend/data/human_review.py) | HITL `PendingHumanReview` 读写 + auto-approve                              |
| [`platform_cost.py`](../backend/data/platform_cost.py) | `PlatformCostLog` 写入 + 成本聚合查询                                    |
| [`push_subscription.py`](../backend/data/push_subscription.py) | Web Push 订阅 CRUD（含端点白名单与 SSRF 防护）                  |
| [`push_sender.py`](../backend/data/push_sender.py) | 真正发推送                                                                  |
| [`workspace.py`](../backend/data/workspace.py) | 用户工作区文件元数据                                                         |
| [`tally.py`](../backend/data/tally.py) | Tally 表单回调                                                                       |
| [`understanding.py`](../backend/data/understanding.py) | Copilot 用户业务理解                                                       |
| [`analytics.py`](../backend/data/analytics.py) | 用户行为统计                                                                  |
| [`diagnostics.py`](../backend/data/diagnostics.py) | 系统诊断（连接 / queue 深度 / 错误率）                                   |
| [`generate_data.py`](../backend/data/generate_data.py) | 测试种子数据                                                                |

### 2.3 模型与共享 Pydantic

| 文件                                  | 内容                                                                              |
| ------------------------------------- | --------------------------------------------------------------------------------- |
| [`model.py`](../backend/data/model.py) | snake_case 应用层模型：`User`、`GraphInput`、`NodeExecutionStats`、`CredentialsMetaInput` 等 |
| [`partial_types.py`](../backend/data/partial_types.py) | Prisma `partial_type_generator` 的输出（自动生成）                  |
| [`includes.py`](../backend/data/includes.py) | Prisma include 树（避免 N+1）                                                |
| [`dynamic_fields.py`](../backend/data/dynamic_fields.py) | 动态 schema 字段处理（Block 输入/输出可变 pin）                  |

## 3. 关键设计

### 3.1 在线/离线 import 路由

`db_accessors.py` 决定一个调用是直连 Prisma 还是走 RPC：

```python
# 直连 Prisma（只在 DatabaseManager 进程里）
from backend.data.graph import save_graph, get_graph

# RPC 客户端（其它进程使用）
from backend.data.db_manager import graph_db
graph = graph_db().get_graph(...)
```

`graph_db()` / `execution_db()` 等返回的是 RPC stub，方法签名与 `graph.py` 一致。

> 这种设计意味着：**`backend/data/graph.py` 里定义的每个公开方法，都自动成为可远程调用的接口。**

### 3.2 事务

```python
from backend.data.db import transaction

async with transaction(timeout=30) as tx:
    await tx.agentgraph.create(...)
    await tx.agentnode.create_many(...)
```

- `timeout` 防止长事务卡死连接池；
- 退出时若抛异常会 rollback，正常退出 commit；
- `save_graph()` 用此机制原子化创建 graph + 节点 + 链接。

### 3.3 Graph 持久化（`graph.py`）

`save_graph(graph)` 内部：
1. 开事务；
2. upsert `AgentGraph(id, version)`；
3. 删除旧 `AgentNode` / `AgentNodeLink`；
4. 批量插入新 `AgentNode`；
5. 批量插入新 `AgentNodeLink`（依赖第 4 步生成的 node id）；
6. 提交。

`get_graph(id, version)` 用 Prisma `include` 一次性取出节点和链接树，避免多次往返。

### 3.4 执行写入（`execution.py`）

- `create_graph_execution()`：写 `AgentGraphExecution`，初始 `status=QUEUED`；
- `update_node_execution_status()`：改单个 node 状态 + 时间戳；
- `upsert_execution_input/output()`：写流式 I/O。Input 使用复合唯一约束保证幂等；output 允许多条（多值流）。
- 所有写入都通过 `event_bus.py` 发布事件，让 WebsocketServer 推前端。

### 3.5 积分扣费（`credit.py`）

> 详细业务流程见 [16 积分与计费](./16-credit-billing.md)，这里只看代码层：

```python
async def spend_credits(user_id, cost_microdollars, metadata) -> int:
    async with transaction() as tx:
        balance_row = await tx.userbalance.find_unique(where={"id": user_id})
        new_balance = balance_row.balance - cost_microdollars
        if new_balance < 0:
            raise InsufficientBalanceError(...)
        await tx.usebalance.update(...)
        await tx.credittransaction.create(
            data=dict(
                transactionKey=metadata["transactionKey"],
                userId=user_id,
                type="USAGE",
                amount=-cost_microdollars,
                runningBalance=new_balance,
                metadata=metadata,
            )
        )
    if new_balance < LOW_BALANCE_THRESHOLD:
        notify_event_bus.publish(LowBalanceData(...))
    return new_balance
```

要点：
- **同事务**修改 `UserBalance` 与写 `CreditTransaction` —— 防漂移；
- `transactionKey` 是幂等键 —— 重复消息只扣一次；
- 通知通过事件总线广播，不阻塞。

### 3.6 用户缓存

```python
@cached(cache=TTLCache(maxsize=1000, ttl=300))
def get_user_by_id_cache(user_id: str) -> User | None: ...
```

- 1000 条 / 5 分钟；
- 用户更新邮箱 / 通知偏好后必须显式 `invalidate(user_id)`。

### 3.7 凭证存储（与 `backend/integrations/credentials_store.py` 协作）

- 真正的凭证 JSON 加密后写在 `User.integrations`（单字段）；
- `data/integrations.py` 只管 webhook 元数据；凭证读写在 `integrations/credentials_store.py`；
- 鉴别：所有"用户的 OAuth Token / API Key"都在 `User.integrations`，不要写到别处。

### 3.8 通知事件

`notifications.py` 定义"业务事件"：

```python
class NotificationEventModel(BaseModel):
    user_id: str
    type: NotificationType
    data: AgentRunData | ZeroBalanceData | LowBalanceData | ...
    queue_type: QueueType  # IMMEDIATE / BATCH / SUMMARY / BACKOFF / ADMIN
```

`notification_bus.py` 异步发布到 Redis；`NotificationManager` 进程消费 RabbitMQ 真正落到邮件/Push。

## 4. 跨模块流程示例

### 4.1 用户启动一次执行

```
[AgentServer route]
  ├─ permissions: validate_graph_execution_permissions()       (graph.py)
  ├─ insert: create_graph_execution()                          (execution.py)
  ├─ publish: rabbitmq.publish(execution_queue)                (rabbitmq.py)
  └─ return execution_id

[ExecutionManager]
  ├─ get_graph(id, version)                                    (graph.py)
  ├─ for each node:
  │     ├─ update_node_execution_status(QUEUED → RUNNING)      (execution.py)
  │     ├─ run block; upsert_execution_output()                (execution.py)
  │     ├─ spend_credits()                                     (credit.py)
  │     └─ event_bus.publish(NODE_EXEC_UPDATE)                 (event_bus.py)
  └─ update graph status → COMPLETED                           (execution.py)
```

### 4.2 第三方 webhook 触发 preset 执行

```
POST /webhooks/{webhook_id}
  ├─ load IntegrationWebhook                                   (integrations.py)
  ├─ verify signature (provider 实现)
  ├─ event_bus.publish(webhook_event)                          (event_bus.py)
[wait_for_webhook_event consumer]
  ├─ enumerate AgentPreset bound to this webhook
  ├─ for each preset: create_graph_execution(...)              (execution.py)
  └─ rabbitmq.publish(execution_queue)
```

## 5. 写代码时的约定

- **新增数据访问函数应放在对应 `*.py`，不要散落到 features 里**；
- 暴露给其它进程的方法**默认就会被 RPC 化**，注意签名要 picklable；
- 写入 + 缓存失效要成对出现；
- 修改 Prisma schema 后必须：
  1. `poetry run prisma generate`；
  2. `poetry run prisma migrate dev`（开发）/ 在迁移目录手写文件（生产）；
  3. `poetry run gen-prisma-stub` 更新类型 stub；
  4. 跑 `poetry run test`。

## 6. 看代码顺序建议

1. `db.py` → 知道连接是怎样建立的；
2. `model.py` → 业务侧 Pydantic 模型是什么样；
3. `graph.py` + `execution.py` → 核心两张图相关；
4. `credit.py` → 学习"事务 + 流水 + 通知"的写法范式；
5. `event_bus.py` + `notification_bus.py` → 学习事件分发；
6. `db_manager.py` → 看 RPC 暴露的入口集合。
