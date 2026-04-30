# 03 进程模型与服务拓扑

## 1. 9 个 AppProcess

后端不是单体进程。`backend/app.py:34` 的 `main()` 会按以下顺序启动 9 个独立进程，前 8 个后台运行，最后一个（ExecutionManager）在前台运行（便于本地终端 `Ctrl+C` 退出整组）：

| #   | 服务                       | 类                                            | 容器/端口        | Entry point          |
| --- | -------------------------- | --------------------------------------------- | ---------------- | -------------------- |
| 1   | DatabaseManager            | `backend.data.db_manager.DatabaseManager`     | `database_manager :8005` | `db`            |
| 2   | Scheduler                  | `backend.executor.Scheduler`                  | `scheduler_server :8003` | `scheduler`     |
| 3   | NotificationManager        | `backend.notifications.NotificationManager`   | `notification_server`    | `notification`  |
| 4   | PlatformLinkingManager     | `backend.platform_linking.manager.PlatformLinkingManager` | （仅一键启动）   | `platform-linking-manager` |
| 5   | WebsocketServer            | `backend.api.ws_api.WebsocketServer`          | `websocket_server :8001` | `ws`            |
| 6   | AgentServer                | `backend.api.rest_api.AgentServer`            | `rest_server :8006`      | `rest`          |
| 7   | ExecutionManager           | `backend.executor.ExecutionManager`           | `executor :8002`         | `executor`      |
| 8   | CoPilotExecutor            | `backend.copilot.executor.manager.CoPilotExecutor` | `copilot_executor :8008` | `copilot-executor`  |
| 9   | CoPilotChatBridge          | `backend.copilot.bot.app.CoPilotChatBridge`   | （仅一键启动）   | `copilot-bot`        |

> Docker 部署时每个 `[tool.poetry.scripts]` 单独跑在一个容器里，对应 `docker-compose.platform.yml` 的服务（如 `rest_server`、`executor`）。`docker-compose.platform.yml` 没有 `platform_linking` 与 `copilot_bot`，这两个服务通常是可选的、按需启用。

## 2. AppProcess / AppService 基础

源码：`backend/util/process.py`、`backend/util/service.py`。

```
class AppProcess:
    service_name: ClassVar[str]
    def run(self) -> None: ...                      # 子类实现工作循环
    def start(self, *, background: bool, **kwargs)  # fork 子进程
    def stop(self) -> None                          # SIGTERM
    def cleanup(self) -> None                       # 信号处理时调用

class AppService(AppProcess):
    """暴露 Pyro5 RPC 端点的进程基类。"""
    @expose      # 装饰器把方法注册到 Pyro5
    def some_method(...): ...
```

`run_processes()`（`backend/app.py:14`）负责：
1. 按顺序 `start(background=True)` 启动前 N-1 个；
2. `start(background=False)` 同步运行最后一个（典型阻塞循环）；
3. `finally` 反向 `stop()` 所有进程，做优雅关闭。

`AppProcess` 还会：
- 拦截 SIGTERM / SIGINT，调用 `cleanup()`；
- 接入 Sentry，未捕获异常自动上报；
- 给日志加上 `[ServiceName]` 前缀。

## 3. 进程间通信路径

| 通道                       | 谁发                                  | 谁收                                           | 用途                                          |
| -------------------------- | ------------------------------------- | ---------------------------------------------- | --------------------------------------------- |
| **Pyro5 RPC**              | AgentServer / Scheduler / Notification | DatabaseManager（`db_manager`），Notification 等 | 跨进程方法调用（如读写 Graph、查询余额） |
| **RabbitMQ `execution`**   | AgentServer / Scheduler                | ExecutionManager                               | 提交一次 Graph 执行                           |
| **RabbitMQ `copilot_execution`** | API / Copilot bridge             | CoPilotExecutor                                | 提交 Copilot 推理请求                          |
| **RabbitMQ `notifications`** | 任意业务模块                         | NotificationManager                            | 邮件 / 推送                                    |
| **Redis Sharded PubSub**   | ExecutionManager / NotificationManager | WebsocketServer                                | 执行进度 / 通知事件 → 推前端                   |
| **Redis 普通 KV**          | 任意                                  | 任意                                           | 缓存、限流、分布式锁、Copilot 流式输出 buffer  |

> RPC 主要用于"小、同步、强一致"的查询；执行/通知这类高吞吐、可异步的业务统一走 MQ。

## 4. 服务职责详解

### 4.1 DatabaseManager
- 文件：`backend/data/db_manager.py`
- 作用：进程内承载 Prisma client 与 RPC 暴露的"业务数据库 API"，让其他无 DB 直连的轻量进程通过 RPC 读写数据。
- 关键 RPC：`graph_db()`、`execution_db()`、`library_db()`、`store_db()`、`chat_db()`、`search()`。
- 启动时：`prisma.connect()` + 预热 `AgentBlock` 元数据。

### 4.2 Scheduler
- 文件：`backend/executor/scheduler.py`
- 框架：APScheduler `BackgroundScheduler` + `SQLAlchemyJobStore`
- 任务：
  - 用户定义的 Graph 定时执行（`AgentExecutionScheduledJob` 表）；
  - 内部周期任务：`accuracy_monitor` / `block_error_monitor` / `late_execution_monitor` / `notification_monitor` 等。
- 落到 DB 的 job 在重启后自动恢复。
- Listener：`job_listener`（成功/失败 Sentry 上报）、`job_missed_listener`（超时告警）、`job_max_instances_listener`（并发越界）。

### 4.3 NotificationManager
- 文件：`backend/notifications/notifications.py`
- 消费 4 个队列（`immediate / summary / batch / failed _v2`）；
- 渲染 Jinja2 模板（`backend/notifications/templates/`）；
- 通过 `postmarker` 发邮件，或调 `pywebpush` 推 Web Push。

### 4.4 PlatformLinkingManager
- 文件：`backend/platform_linking/manager.py`
- 作用：在 Discord / Telegram / Slack 中将平台账号与 AutoGPT 用户绑定，并把消息桥接到 Copilot 会话。
- RPC：`resolve_server_link`、`resolve_user_link`、`create_server_link_token`、`start_chat_turn`。

### 4.5 WebsocketServer
- 文件：`backend/api/ws_api.py`
- FastAPI WebSocket，挂载 `/ws/...`；
- 维护 `ConnectionManager`（`backend/api/conn_manager.py`），每个连接订阅 Redis 上对应 channel；
- Sharded PubSub 解决 Redis Cluster 跨槽问题；自带断线重连与 MOVED 处理。

### 4.6 AgentServer
- 文件：`backend/api/rest_api.py`
- FastAPI 主应用 + 挂载 `external_api`；
- `lifespan`：连接 Prisma / Redis / Supabase / 加载 Feature Flag；
- 中间件：`SecurityHeadersMiddleware`（缓存控制 + 安全头）+ CORS + GZip + Prometheus；
- 路由聚合：`backend/api/features/v1.py` 把所有子 router 装配进去。

### 4.7 ExecutionManager
- 文件：`backend/executor/manager.py`
- 启动一个 ThreadPoolExecutor（大小由 `num_graph_workers` 控制）；
- 同步消费 `execution_run_queue` + `execution_cancel_queue`；
- 每个任务交给 worker 线程的 `ExecutionProcessor` 跑完整张图（详见 [11](./11-execution-engine.md)）。

### 4.8 CoPilotExecutor
- 文件：`backend/copilot/executor/manager.py`
- 单独消费 `copilot_execution_queue`，避免 Copilot 长任务把图执行 worker 堵死；
- 每次会话用独立 `CoPilotProcessor` 处理一个 turn；
- 双重锁：`refresh lock`（防 Copilot 凭证被并发刷新）+ `acquire lock`（防 in-flight token 失效）。

### 4.9 CoPilotChatBridge
- 文件：`backend/copilot/bot/app.py`
- 是个 RPC + Discord/Telegram bot 适配器；
- 把外部消息塞进 `pending_messages` Redis 队列，再触发 Copilot 任务；
- 同时把 Copilot 流式输出广播回各个适配器。

## 5. 端口速查

| 端口   | 服务               | 协议       |
| ------ | ------------------ | ---------- |
| 8001   | WebsocketServer    | WebSocket  |
| 8002   | ExecutionManager   | RPC + 健康检查 |
| 8003   | Scheduler          | RPC        |
| 8005   | DatabaseManager    | RPC        |
| 8006   | AgentServer        | HTTP REST  |
| 8008   | CoPilotExecutor    | RPC        |
| 17000~17002 | Redis Cluster shards | Redis  |
| 5672   | RabbitMQ           | AMQP       |
| 6380   | FalkorDB           | Redis 协议 |
| 3310   | ClamAV             | clamd      |

> 端口可由环境变量覆盖：`AGENTSERVER_PORT`、`SCHEDULER_PORT`、`EXECUTIONMANAGER_PORT` 等。Docker 部署使用 service 名（`rest_server`、`scheduler_server` 等）作为 host。

## 6. 启动顺序与依赖

```
PostgreSQL ──┐
Redis ──────┤
RabbitMQ ───┘
   │
   ▼
migrate (one-shot, prisma migrate deploy)
   │
   ▼
DatabaseManager ─→ Scheduler / Notification / WebsocketServer / AgentServer / ExecutionManager / CoPilotExecutor
                                                                 (互相独立，可并行)
```

`docker-compose.platform.yml` 的 `depends_on: condition: service_healthy/completed_successfully` 强制了这个顺序。

## 7. 关停与重启

- `app.py` 的 `finally` 反序停止：先停 Copilot bot → ... → 最后停 DatabaseManager。
- 单独跑容器时，每个进程独立处理 SIGTERM；
- ExecutionManager 收到 SIGTERM 后会：
  1. 取消 RabbitMQ basic_consume；
  2. 等待当前 Graph 执行完成（有超时），未完成的会被消息 nack 回队；
  3. 关闭线程池；
  4. 释放 Redis 集群锁。

## 8. 一键启动（开发）

```bash
# 装依赖
poetry install

# 起基础设施
docker compose up -d

# 跑迁移
poetry run prisma migrate dev

# 一次起 9 个进程（看着 ExecutionManager 的日志）
poetry run app
```

跑通后访问：
- REST: http://localhost:8006/docs（Swagger）
- WebSocket: ws://localhost:8001/ws/{exec_id}?token=...
- Prometheus 指标: http://localhost:8006/metrics
