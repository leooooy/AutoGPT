# 01 架构总览

## 1. 一句话概括

AutoGPT Platform 后端是 **多进程 + 异步消息驱动** 的 Agent 工作流平台：以 PostgreSQL 为持久化层，Redis 提供事件总线/缓存/集群锁，RabbitMQ 解耦执行队列与通知队列，Pyro5 RPC 串联进程间调用；REST 与 WebSocket 由独立进程提供，执行引擎、Copilot、调度器、通知、Bot 桥接也各自独立部署，通过 `backend/app.py` 的 `run_processes()` 在本地一键拉起。

## 2. 分层视图

```
┌──────────────────────────────────────────────────────────────────┐
│                    前端 / SDK / 第三方 Webhook                   │
└────┬───────────────┬───────────────┬──────────────────────┬──────┘
     │ HTTP/REST     │ WebSocket     │ External v1 (API Key)│ Webhook
     ▼               ▼               ▼                      ▼
 ┌────────────────────────────────────────────────────────────────┐
 │  接入层 (FastAPI)                                              │
 │  - AgentServer (REST, :8006)        - WebsocketServer (:8001)  │
 │  - external_api app (mounted)       - middleware (security)    │
 └─────┬──────────────────────────────────────────────────────────┘
       │ Prisma RPC / RabbitMQ publish / Redis publish
       ▼
 ┌────────────────────────────────────────────────────────────────┐
 │  业务/执行层                                                   │
 │  - DatabaseManager (db RPC, :8005)                             │
 │  - ExecutionManager (consume execution_queue, :8002)           │
 │  - Scheduler (APScheduler, :8003)                              │
 │  - NotificationManager (consume notification queues)           │
 │  - PlatformLinkingManager (Discord/Slack 链接)                 │
 │  - CoPilotExecutor / CoPilotChatBridge (:8008)                 │
 └─────┬──────────────────────────────────────────────────────────┘
       │
       ▼
 ┌────────────────────────────────────────────────────────────────┐
 │  数据/基础设施                                                 │
 │  PostgreSQL + pgvector | Redis Cluster (3 shards) | RabbitMQ   │
 │  ClamAV  | Supabase Auth | Google Cloud Storage | FalkorDB     │
 └────────────────────────────────────────────────────────────────┘
```

## 3. 核心模块拓扑

```
backend/
├── app.py                # 单机一键启动所有 AppProcess
├── rest.py / ws.py / scheduler.py / exec.py / db.py / notification.py / ...
│                         # 各服务的独立 entry point（容器内单进程使用）
├── api/                  # 接入层（HTTP / WebSocket）
│   ├── rest_api.py       # AgentServer
│   ├── ws_api.py         # WebsocketServer
│   ├── conn_manager.py   # WebSocket 订阅 + Redis Pub/Sub 桥接
│   ├── external/         # API Key 鉴权的对外 v1 API
│   ├── features/         # 按业务子域拆分的路由群
│   ├── middleware/       # 安全头、缓存控制
│   └── model.py          # WSMessage / CreateGraph 等共享 schema
├── data/                 # 数据访问层（Prisma + Redis + RabbitMQ）
│   ├── db.py / db_manager.py
│   ├── graph.py          # AgentGraph CRUD（核心）
│   ├── execution.py      # GraphExecution / NodeExecution 读写
│   ├── credit.py         # 积分流水
│   ├── integrations.py   # Webhook & Credentials 元数据
│   ├── notifications.py / notification_bus.py
│   ├── event_bus.py      # Redis Sharded Pub/Sub
│   ├── rabbitmq.py / redis_client.py
│   └── ...
├── executor/             # 执行引擎与计费
│   ├── manager.py        # ExecutionManager + ExecutionProcessor
│   ├── scheduler.py      # APScheduler + SQLAlchemyJobStore
│   ├── cost_tracking.py / billing.py
│   └── cluster_lock.py
├── blocks/               # 所有 Block 实现（100+ 文件 / 子目录）
│   ├── _base.py          # Block / BlockSchema 基类
│   ├── llm.py / http.py / agent.py / branching.py / ...
│   └── github/, google/, notion/, ...   # 第三方集成 Block
├── sdk/                  # ProviderBuilder / 注册表
├── integrations/         # Credentials / OAuth Handler / Webhook Manager
├── notifications/        # NotificationManager 进程 + 邮件模板
├── copilot/              # Copilot AI 助手（双执行链 + Bot 桥接）
├── monitoring/           # Prometheus 指标 / Late execution monitor / ...
├── platform_linking/     # Discord/Slack 账号链接进程
└── util/                 # 通用工具（process / service / cache / encryption / ...）
```

## 4. 主链路：一次 Graph 执行的端到端流程

```
1. [前端]
       POST /api/graphs/{id}/execute/{version}
                │
2. [AgentServer]
       authenticate (JWT) → validate permission → 写入
       AgentGraphExecution (status=QUEUED)
                │
       publish 到 RabbitMQ exchange `execution`
                │
3. [ExecutionManager 进程]
       _consume_execution_run() 取出消息
                │
       提交给 ThreadPoolExecutor (num_graph_workers)
                │
       ExecutionProcessor.on_graph_execution()
         ├─ 加载 GraphModel（节点/连接/输入）
         ├─ 收集 in_degree=0 的起始节点入队
         ├─ 并发执行 ExecutionProcessor.on_node_execution(node):
         │     ├─ 验证 input
         │     ├─ creds_manager.acquire(user_id, cred_id)  # 含分布式锁
         │     ├─ block.execute(input_data, **kwargs)
         │     │      └─ async generator yield (output_name, value)
         │     ├─ 写入 AgentNodeExecutionInputOutput
         │     ├─ 计费: cost_tracking + credit.spend_credits()
         │     ├─ 发布事件到 Redis (NODE_EXEC_UPDATE)
         │     └─ register_next_executions() 沿 link 推进
         │
         └─ 全部完成 → 写 GraphExecution.stats、status=COMPLETED
                │
4. [WebsocketServer]
       SUBSCRIBE 收到 Redis 事件 → 推 WSMessage 给前端
                │
5. [NotificationManager]
       消费 immediate_notifications_v2 → 邮件 / Web Push
```

整条链路上每一段都对应一个独立进程，用 RabbitMQ / Redis Pub/Sub 解耦，单点故障不会拖垮全链路（执行队列堆积仍可读，WebSocket 断线自动重连）。

## 5. 关键设计取舍

| 取舍                    | 选择                                       | 原因                                                                 |
| ----------------------- | ------------------------------------------ | -------------------------------------------------------------------- |
| 进程模型                | 多进程（一个服务一个进程）                 | CPU 密集型 Block 可水平扩展；单服务挂掉不影响其他                    |
| 进程间通信              | Pyro5 RPC + RabbitMQ + Redis Pub/Sub       | RPC 用于配置类调用，队列承载执行/通知流，Redis 承载事件广播          |
| ORM                     | Prisma (`prisma-client-py`)                | TypeScript 风格 schema、类型生成、Postgres 视图 / 全文 / pgvector 一站支持 |
| 队列模式                | RabbitMQ Quorum Queue                      | 比 Classic Mirrored 更容错；执行任务 ack=manual 防止丢任务           |
| 缓存 + 事件             | Redis Cluster（3 主分片）+ Sharded PubSub  | Sharded PubSub 解决 Cluster 模式下普通 PubSub 跨槽位限制             |
| 鉴权                    | Supabase JWT + 自家 API Key                | Supabase 处理用户管理；API Key 给 SDK / Webhook 使用                 |
| Block 注册              | 启动时扫描 `backend/blocks/` 目录          | 零配置新增 Block；同步元数据到 `AgentBlock` 表                       |
| 凭证存储                | 加密 JSON 存于 `User.integrations`         | 单查询拿全部用户凭证；加密 + 锁双重保护                              |
| 计费                    | 微美元 (microdollars) + CreditTransaction  | 避免浮点；流水表可审计；UserBalance 表作为快照                       |
| 病毒扫描                | ClamAV (异步 aioclamd) + 上传时拦截        | 文件上传/store 媒体素材必经                                          |

## 6. 数据存储分布

| 存储                | 主要内容                                                                                       | 模块入口                                  |
| ------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------- |
| PostgreSQL          | 所有持久化业务数据（用户、Graph、执行、凭证、商城、通知、积分、Copilot 会话）                  | `backend/data/db.py`                      |
| Redis Cluster       | 执行事件总线、Web 缓存、限流计数器、Copilot 流式输出、分布式锁、订阅状态                       | `backend/data/redis_client.py`            |
| RabbitMQ            | `execution`（图执行任务）、`notifications`（邮件/Push）、`copilot_execution`                   | `backend/data/rabbitmq.py`                |
| FalkorDB            | Copilot Graphiti 时序记忆图（向量 + 关系）                                                     | `backend/copilot/graphiti/`               |
| Google Cloud Storage| 文件 / 工作区文件 / Store 媒体素材                                                             | `backend/util/gcs_utils.py`               |
| Supabase            | 用户认证、邮件模板（部分）、Realtime（不被 Backend 主动用）                                    | `backend/util/clients.py`                 |

## 7. 后端代码量级（约）

```
后端 Python 代码    ~12 万行
Prisma schema       ~2000 行
Block 实现          50+ 子目录 / 100+ 文件
测试                colocated `*_test.py`，覆盖几乎每个数据/路由模块
```

## 8. 进一步阅读

- 各服务进程角色 → [03 进程模型](./03-process-model.md)
- 表结构 → [05 数据模型](./05-data-model.md)
- 执行细节与时序 → [11 执行引擎](./11-execution-engine.md)
- 一次 HTTP 请求穿过哪些中间件 → [07 REST API 总览](./07-api-rest.md)
