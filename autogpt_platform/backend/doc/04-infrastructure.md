# 04 基础设施依赖

## 1. 概览

| 基础设施            | 形态                                  | 主要用途                                               | 客户端模块                              |
| ------------------- | ------------------------------------- | ------------------------------------------------------ | --------------------------------------- |
| PostgreSQL + pgvector | 单实例（本地 / 云）                  | 业务数据持久化；向量索引                               | `backend/data/db.py`                    |
| Redis Cluster       | 3 主分片（本地 docker compose）       | 事件总线 / 缓存 / 限流 / 分布式锁 / 流式输出 buffer    | `backend/data/redis_client.py`          |
| RabbitMQ            | Quorum Queues                         | 执行队列、通知队列、Copilot 队列、Webhook ingress      | `backend/data/rabbitmq.py`              |
| ClamAV              | 单容器                                | 文件上传病毒扫描                                       | `backend/util/virus_scanner.py`         |
| Supabase            | 自部署或托管                          | 用户鉴权 / Email                                       | `backend/util/clients.py`               |
| Google Cloud Storage| 真 GCS 或本地模拟                     | 媒体素材、用户工作区文件                               | `backend/util/gcs_utils.py`             |
| FalkorDB            | 单容器                                | Copilot Graphiti 时序记忆图（Redis 协议 + 向量）       | `backend/copilot/graphiti/`             |
| Stripe              | 外部 SaaS                             | 订阅 / 充值                                            | `backend/executor/billing.py` 等         |

> 不同部署环境组合略有差异：本地 dev 强依赖 Postgres / Redis / RabbitMQ / ClamAV；GCS 与 Supabase 在没有真实凭证时降级为本地实现或禁用。

## 2. PostgreSQL

### 2.1 连接

- DSN 由 `DATABASE_URL` 决定（`backend/.env.default:16`）：
  ```
  postgresql://postgres:****@db:5432/postgres?schema=platform&connect_timeout=60
  ```
- 业务表全部建在 `platform` schema 下；
- `DIRECT_URL` 用于 Prisma migration 等需要绕过 PgBouncer 的场景。

### 2.2 关键参数（`backend/util/settings.py`）

| 设置                    | 默认 | 含义                       |
| ----------------------- | ---- | -------------------------- |
| `DB_CONNECTION_LIMIT`   | 12   | 单个进程的连接池上限       |
| `DB_CONNECT_TIMEOUT`    | 60s  | 建连超时                   |
| `DB_POOL_TIMEOUT`       | 300s | 取连接超时                 |
| `DB_SCHEMA`             | platform | 业务 schema 名         |

### 2.3 扩展

- `pgvector`：提供向量类型与 ANN 索引；用于 Store hybrid search 与 Copilot 知识库（`UnifiedContentEmbedding` 等表）。
- `fullTextSearch`、`views` 已开启（schema.prisma:12）。
- 视图：`scripts/generate_views.py` 通过 `poetry run analytics-views` 生成分析用视图。

### 2.4 Prisma

- Schema：`backend/schema.prisma`（见 [05](./05-data-model.md)）
- Generator：`prisma-client-py`，异步接口；
- 每次启动后运行：`prisma generate && python scripts/gen_prisma_types_stub.py && prisma migrate deploy`。

## 3. Redis Cluster

### 3.1 拓扑

本地默认起 3 个 Redis 主（`redis-0/1/2`），各自端口 17000/17001/17002。无副本（`--cluster-replicas 0`），定位是缓存。`redis-init` 一次性 sidecar 调用 `redis-cli --cluster create` 完成集群组装；后续 `up` 幂等。

### 3.2 连接

`backend/data/redis_client.py`：
- 自动判断单点 / 集群模式（环境变量 `REDIS_CLUSTER_HOST/PORT`）；
- 关键超时：`SOCKET_TIMEOUT=30s`、`SOCKET_CONNECT_TIMEOUT=5s`、`HEALTH_CHECK_INTERVAL=30s`；
- 启用 `address_remap`，让本地宿主机与容器都能解析到相同的节点；
- 从节点宣告的 hostname（`redis-0/1/2`）出发做 CLUSTER SLOTS。

### 3.3 用途分类

| 用途                  | Key 风格                                  | 写入模块                         |
| --------------------- | ----------------------------------------- | -------------------------------- |
| 执行事件总线          | `execution_event_bus_name/{graph_id}/{exec_id}` | `backend/data/event_bus.py` (Sharded PubSub) |
| 通知事件总线          | `notification_event_bus_name/{user_id}`   | 同上                             |
| Copilot 流式输出      | Redis Streams + Sharded PubSub            | `backend/copilot/stream_registry.py` |
| 限流计数器            | `rate:{user_id}:{bucket}`                 | `backend/copilot/rate_limit.py`  |
| Store agent 缓存      | `store:agent:{username}/{name}`           | `backend/api/features/store/cache.py` |
| 凭证刷新锁            | `lock:cred_refresh:{cred_id}`             | `backend/integrations/creds_manager.py` |
| 集群锁                | `lock:cluster:{key}`                      | `backend/executor/cluster_lock.py` |
| Push 订阅追踪         | `push:subs:{user_id}`                     | `backend/data/push_subscription.py` |

### 3.4 Sharded PubSub

普通 `PUBLISH/SUBSCRIBE` 在 Cluster 模式下只在单节点广播。Backend 统一用 `SSUBSCRIBE`：消息按 channel 哈希到具体 slot 节点。在 `event_bus.py` 中通过自定义 `BaseRedisEventBus[M]` 封装类型化消息发布订阅，具备：
- 大消息 truncation（防 connection 崩溃）；
- 序列化 fallback（Unicode → ASCII）；
- 不支持模式订阅（PSUBSCRIBE 不可用）→ 业务层用确定的 channel 名。

## 4. RabbitMQ

### 4.1 连接

`backend/data/rabbitmq.py`：
- 同步客户端（`pika`）用于业务发送；
- 异步客户端（`aio-pika`）用于消费（执行/通知 worker 内部）；
- 关键参数：`BLOCKED_CONNECTION_TIMEOUT=300s`、`SOCKET_TIMEOUT=30s`、`CONNECTION_ATTEMPTS=5`、`RETRY_DELAY=1s`；
- 用 Exchange/Queue Pydantic 模型描述拓扑，启动时 `declare_topology()`。

### 4.2 拓扑

| Exchange                | 类型  | 队列                              | 消费者                  |
| ----------------------- | ----- | --------------------------------- | ----------------------- |
| `execution`             | TOPIC | `execution_run_queue` / `execution_cancel_queue` | ExecutionManager      |
| `copilot_execution`     | TOPIC | `copilot_execution_queue`         | CoPilotExecutor         |
| `notifications`         | TOPIC | `immediate_notifications_v2`、`summary_notifications_v2`、`batch_notifications_v2`、`failed_notifications_v2` | NotificationManager |
| `webhook`（按需）        | TOPIC | provider 维度                     | webhook 处理            |

所有队列默认 Quorum 模式：
- 多副本（生产建议 ≥3 节点 RabbitMQ 集群）；
- 写入 disk 持久化；
- `manual ack`，避免 worker 崩溃时丢任务。

## 5. ClamAV

- 服务：`docker-compose.yml:112`（端口 3310）；
- 客户端：`aioclamd` 异步；
- 触发点：`store_media_file()`（`backend/util/file.py`）—— 上传 / 拉取媒体前先 `INSTREAM` 扫描，发现感染抛错并不持久化。
- 适用范围：Library 上传文件、Store 商城素材、Workspace 文件。

## 6. Supabase

- 用途：作为用户认证与基础邮件后端（部分场景）；
- 客户端：`supabase 2.28.0`；通常以 Service Role Key 调；
- JWT 验证：`autogpt_libs.auth.jwt_utils.get_jwt_payload()`，`JWT_VERIFY_KEY` 必须与 Supabase 一致；
- 开发/CI 可关：`ENABLE_AUTH=false`，此时 `DEFAULT_USER_ID` 直接被注入。

## 7. Google Cloud Storage

- 用途：Store 媒体（封面图）、用户工作区文件、临时素材；
- 客户端：`gcloud-aio-storage`（异步）+ `google-cloud-storage`（同步）；
- 配置：`MEDIA_GCS_BUCKET_NAME`；
- 缺省时：本地实现存到 `workspaces/` volume。

## 8. FalkorDB

- 给 Copilot 的 Graphiti 时序记忆图用；
- 端口：6380（Redis 协议）+ 3001（Web UI）；
- LaunchDarkly Flag `graphiti-memory` 控制是否启用；
- 配置：`GRAPHITI_FALKORDB_HOST/PORT/PASSWORD`、`GRAPHITI_LLM_MODEL`、`GRAPHITI_EMBEDDER_MODEL`。

## 9. Stripe

- 用途：充值（top-up）、订阅扣费（PRO / MAX / BUSINESS / ENTERPRISE）、定价同步；
- API key：`STRIPE_API_KEY`（不在 `.env.default` 默认值中）；
- Webhook：`POST /api/credits/stripe_webhook`；
- 业务侧：`backend/data/credit.py`、`backend/executor/billing.py`。

## 10. 配置体系

- 配置加载顺序（`backend/util/settings.py`）：
  1. `backend/.env.default`（默认值，git 跟踪）
  2. `backend/.env`（用户覆盖，gitignored）
  3. 环境变量（最高优先级）
- 类：`Config(BaseSettings)`，按域分组（DB / Redis / RabbitMQ / 鉴权 / Pyro / 信用 / Copilot 等）；
- 100+ 字段，全部带类型校验；
- 修改 `.env.default` 等同于改全局默认，必须慎重 review。

## 11. 健康检查

- `/health`、`/_next/static/health`、`/static/health` 等被 `SecurityHeadersMiddleware` 列入允许缓存白名单；
- `docker-compose.platform.yml` 为每个外部服务定义了 `healthcheck`；启动顺序由 `depends_on: condition: service_healthy` 强制；
- Prometheus `/metrics` 暴露 readiness 数据。
