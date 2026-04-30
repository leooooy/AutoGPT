# 22 部署

## 1. 生产拓扑

后端是多进程系统，生产环境**每个 AppProcess 独立部署到容器**：

```
              ┌─ rest_server (N 副本)         ──┐
              ├─ websocket_server (N 副本)    ──┤── LB / Ingress
              │
              ├─ executor (N 副本，水平扩)
              ├─ scheduler (单实例！)
              ├─ database_manager (1~少数副本)
              ├─ notification_server (少数副本)
              ├─ copilot_executor (N 副本)
              ├─ copilot_bot (单实例 / per-bot)
              └─ platform_linking (单实例)
                            │
                            ▼
            ┌─ PostgreSQL（带 pgvector）
            ├─ Redis Cluster
            ├─ RabbitMQ Cluster
            ├─ ClamAV
            ├─ Supabase
            ├─ FalkorDB
            ├─ Stripe
            └─ GCS
```

> **Scheduler 必须单实例**：APScheduler 默认非分布式，多实例会让一个 cron 任务被多次触发。需要 HA 时引入 Redis 锁做 leader election。
>
> Bot/PlatformLinking 也建议单实例（适配器侧每个 bot 通常是单连接）。

## 2. Docker 镜像

### 2.1 Dockerfile

[`autogpt_platform/backend/Dockerfile`](../Dockerfile) 多阶段构建：

| stage     | 用途                                            |
| --------- | ----------------------------------------------- |
| `base`    | 安装系统依赖、Poetry                            |
| `builder` | 装 Python 依赖（poetry install）                |
| `migrate` | 仅含 prisma + 迁移文件（一次性容器）            |
| `server`  | 跑 backend.* 服务的运行时镜像                   |

每个进程容器都是同一个 `server` 镜像，只是 `command` 不同：

```yaml
# 各容器 entrypoint（在 docker-compose.platform.yml 里）
command: ["rest"]            # rest_server
command: ["ws"]              # websocket_server
command: ["executor"]        # executor
command: ["scheduler"]       # scheduler_server
command: ["db"]              # database_manager
command: ["notification"]    # notification_server
command: ["python", "-u", "-m", "backend.copilot.executor"]
```

`rest` / `ws` / ... 这些名字对应 `pyproject.toml [tool.poetry.scripts]` 里的入口。

## 3. docker-compose

仓库提供 3 个 compose：

| 文件                                  | 内容                                    |
| ------------------------------------- | --------------------------------------- |
| `autogpt_platform/docker-compose.yml` | 顶层；用 `extends` 引用 `platform.yml`  |
| `autogpt_platform/docker-compose.platform.yml` | 后端各进程容器定义                |
| `autogpt_platform/db/docker/docker-compose.yml` | Supabase（开发用）                |
| `autogpt_platform/backend/docker-compose.test.yaml` | 测试基础设施（独立 PG）         |

### 3.1 启动顺序

`migrate` → `database_manager` → 其余服务，由 `depends_on: condition: service_healthy/completed_successfully` 强制：

```
db (postgres) ──► migrate ──┬──► database_manager ──┬──► rest_server
                              │                      ├──► executor
redis-0 healthy ──────────────┤                      ├──► websocket_server
rabbitmq healthy ─────────────┘                      ├──► scheduler_server
                                                     ├──► notification_server
                                                     └──► copilot_executor
```

### 3.2 环境变量加载顺序

平台约定：

1. `backend/.env.default`（默认值，git 跟踪）；
2. `backend/.env`（用户覆盖，gitignored）；
3. compose 的 `environment:` 块（容器级覆盖，docker-compose.platform.yml 里把 host 改成 service 名）；
4. shell `export`；
5. `docker compose run -e VAR=value`。

注意：`docker-compose.platform.yml` 没有 `${VARIABLE}` 替换，所有默认值都硬编码 —— 因此 compose 文件能独立工作，不依赖外部变量。

### 3.3 公共环境锚点

```yaml
x-backend-env: &backend-env
  PYRO_HOST: "0.0.0.0"
  AGENTSERVER_HOST: rest_server
  SCHEDULER_HOST: scheduler_server
  DATABASEMANAGER_HOST: database_manager
  EXECUTIONMANAGER_HOST: executor
  NOTIFICATIONMANAGER_HOST: notification_server
  CLAMAV_SERVICE_HOST: clamav
  DB_HOST: db
  REDIS_HOST: redis-0
  REDIS_PORT: "17000"
  REDIS_USE_ANNOUNCED_ADDRESS: "true"
  RABBITMQ_HOST: rabbitmq
  GRAPHITI_FALKORDB_HOST: falkordb
  GRAPHITI_FALKORDB_PORT: "6379"
  SUPABASE_URL: http://kong:8000
  DATABASE_URL: postgresql://...@db:5432/postgres?...
```

YAML 锚点把这套东西注入每个容器，省去重复。

### 3.4 端口映射

| 服务               | 容器端口 | 宿主端口 |
| ------------------ | -------- | -------- |
| rest_server        | 8006     | 8006     |
| executor           | 8002     | 8002     |
| websocket_server   | 8001     | 8001     |
| scheduler_server   | 8003     | 8003     |
| database_manager   | 8005     | 8005     |
| copilot_executor   | 8008     | 8008     |
| redis-0/1/2        | 17000/01/02 | 同 |
| rabbitmq           | 5672     | 5672     |
| falkordb           | 6379/3000| 6380/3001|
| clamav             | 3310     | 3310     |

## 4. 迁移策略

### 4.1 dev

```bash
poetry run prisma migrate dev      # 自动创建迁移文件 + 应用
```

### 4.2 prod

```bash
poetry run prisma migrate deploy   # 仅按已存在的迁移文件 apply（不会生成新文件）
```

`migrate` 容器在 `compose up` 时跑 `prisma generate && python scripts/gen_prisma_types_stub.py && prisma migrate deploy`。状态健康检查 = `prisma migrate status | grep "No pending migrations"`。

### 4.3 zero-downtime 迁移规范

- 加列 → 默认值要安全（不要 NOT NULL + 无默认）；
- 删列 → 分两次：先停用代码引用（部署 N），再删（部署 N+1）；
- 改列名 → 先 add new column + 双写，再迁，再 drop；
- 加索引 → `CONCURRENTLY`；Prisma 不直接支持 → 手写 SQL migration。

## 5. 水平扩容指南

| 服务                  | 扩容收益                            | 注意                                                  |
| --------------------- | ----------------------------------- | ----------------------------------------------------- |
| `rest_server`         | 高 QPS                              | 无状态，前置 LB                                       |
| `websocket_server`    | 长连接数                            | 无状态；可 sticky session 减少订阅迁移                |
| `executor`            | 图执行吞吐                          | 无状态；RabbitMQ 自动分发；多实例间通过 cluster_lock 协调 |
| `database_manager`    | RPC 吞吐                            | 多实例时 RPC 客户端要 round-robin                     |
| `notification_server` | 邮件 / Push 吞吐                    | 多实例 OK；幂等基于 transactionKey                    |
| `copilot_executor`    | Copilot 吞吐                        | 多实例 OK                                             |
| `scheduler_server`    | **不要扩**                          | 多实例会重复触发                                      |
| `copilot_bot`         | 不能简单扩                           | bot 适配器单连接为主                                  |

## 6. 部署前 checklist

- [ ] 数据库做了备份；
- [ ] 新迁移在 staging 跑通；
- [ ] `ENCRYPTION_KEY` 没变（变 = 凭证全失效）；
- [ ] `JWT_VERIFY_KEY` 与 Supabase 一致；
- [ ] `VAPID_PRIVATE_KEY/PUBLIC_KEY/CLAIM_EMAIL` 是真实生产密钥；
- [ ] `STRIPE_API_KEY` / webhook secret 切换到 live；
- [ ] `MEDIA_GCS_BUCKET_NAME` 指向生产桶 + service account key 已挂；
- [ ] `SENTRY_DSN` 区分环境；
- [ ] `LANGFUSE_*` 指向生产；
- [ ] LaunchDarkly Flag 已 review；
- [ ] Discord 告警 webhook 不指向测试群。

## 7. 升级流程

```
1. PR merge to dev → CI 全绿
2. 部署 staging（auto from dev）
3. 跑 smoke test + 手动验关键路径
4. PR dev → master
5. 部署 prod
   a. migrate 容器先跑（迁移）
   b. database_manager 滚动更新
   c. 业务进程滚动更新（rest / executor / ws / ...）
   d. scheduler 单实例重启（短暂窗口可接受）
6. 观察 Sentry / Prometheus / 队列深度 30 分钟
```

## 8. 灰度

- LaunchDarkly Flag 是首选灰度手段（按 user / 按比例）；
- 进程级灰度用副本数（少数实例先升级）；
- API 兼容性：新增字段 / 端点向后兼容；删除前先发出 deprecation 通告 + 监控调用方。

## 9. 数据备份

- PG：每日全量 + WAL 归档；
- 关键表（`User`、`UserBalance`、`CreditTransaction`、`AgentGraph`）做 point-in-time recovery；
- Redis 是缓存，**可以丢**；FalkorDB 同；
- RabbitMQ Quorum Queue 数据是消息流，宕机重启可恢复未 ack 消息；
- GCS 文件一般 versioning 已开。

## 10. 灾备 / 演练

- 建议 quarter 一次：
  - 拉一份生产 dump → restore 到 staging；
  - 重放最近 24h Stripe webhook（用 Stripe CLI）；
  - 关停一个 RabbitMQ 节点观察消费者切换；
  - 关停 Redis 一个 shard 观察 cluster 自愈与连接重连。

## 11. 与前端协调

- API schema 改动 → `poetry run export-api-schema` → 提交 OpenAPI；
- 前端 `pnpm generate:api` 重生成；
- 不能在一次部署里既改后端响应又依赖前端用新字段 —— 兼容窗口至少一次部署。

## 12. 进一步阅读

- [`backend/AGENTS.md`](../AGENTS.md) —— 后端贡献规范；
- [`autogpt_platform/AGENTS.md`](../../AGENTS.md) —— 仓库级 PR / 分支策略；
- 仓库 `docs/platform/` —— 平台架构文档。
