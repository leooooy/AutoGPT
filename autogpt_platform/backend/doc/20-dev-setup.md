# 20 本地开发与启动

## 1. 前置依赖

| 工具             | 推荐版本             | 说明                                                  |
| ---------------- | -------------------- | ----------------------------------------------------- |
| Python           | 3.10–3.13            | `pyproject.toml: python = ">=3.10,<3.14"`             |
| Poetry           | **2.2.1**（pinned）  | 不要自动升级，先确认 dependabot 兼容                   |
| Docker / Compose | 最新                 | 启动 PG / Redis / RabbitMQ / ClamAV / FalkorDB        |
| Node.js          | LTS                  | 仅前端需要；后端单独跑可不装                           |
| Git              | 任意                 | 仓库需 git 元数据                                     |

可选：`pre-commit`（仓库根有钩子）。

## 2. 拉代码

```bash
git clone https://github.com/Significant-Gravitas/AutoGPT.git
cd AutoGPT/autogpt_platform/backend
```

## 3. 安装依赖

```bash
poetry install
```

首次会装 200+ 包，时间较长。

## 4. 配置 `.env`

```bash
cp .env.default .env
# 必要时改：DB / Redis / RabbitMQ host & 密钥
```

最少改的项（如果跟默认 docker compose 拓扑不一样）：

- `DB_HOST` / `DB_PORT`
- `REDIS_HOST` / `REDIS_PORT`
- `SUPABASE_URL` / `SUPABASE_SERVICE_ROLE_KEY` / `JWT_VERIFY_KEY`
- `ENCRYPTION_KEY`、`UNSUBSCRIBE_SECRET_KEY`、`VAPID_*`

> **`ENCRYPTION_KEY` 必须保密**：它解密 `User.integrations` 里所有第三方凭证，泄漏 = 用户凭证全军覆没。

可关闭项（开发友好）：

- `ENABLE_AUTH=false` —— 跳过 JWT 校验，直接用 `DEFAULT_USER_ID`；
- `ENABLE_CREDIT=false` —— 跳过扣费检查；
- `MEDIA_GCS_BUCKET_NAME=` —— 空则用本地 workspaces。

## 5. 启动基础设施

仓库根目录（`autogpt_platform/`）：

```bash
cd ..
docker compose up -d        # 起 PG / Redis Cluster / RabbitMQ / ClamAV / FalkorDB / Supabase
```

启动顺序由 `depends_on: condition` 控制；可用 `docker compose ps` 看状态。

## 6. 数据库迁移

```bash
cd backend
poetry run prisma generate                  # 生成 Python 类型
poetry run python scripts/gen_prisma_types_stub.py  # 类型 stub
poetry run prisma migrate dev               # 应用迁移并生成新迁移（dev）
# 或者纯应用：
poetry run prisma migrate deploy
```

> Docker 模式下有专门的 `migrate` 容器在 `compose up` 时自动跑 `prisma migrate deploy`。

## 7. 一键起后端（9 进程）

```bash
poetry run app
```

终端输出会带 `[ServiceName]` 前缀，可以按服务追日志。前 8 个后台跑，最后一个（ExecutionManager）前台跑 —— `Ctrl+C` 会触发 `run_processes()` 的 `finally` 反序停止所有进程。

## 8. 单进程跑（调试）

某个服务挂了想单独 debug：

```bash
poetry run rest                       # 仅 REST
poetry run ws                         # 仅 WebSocket
poetry run executor                   # 仅 ExecutionManager
poetry run scheduler                  # 仅 Scheduler
poetry run notification               # 仅 NotificationManager
poetry run db                         # 仅 DatabaseManager
poetry run platform-linking-manager
poetry run copilot-executor
poetry run copilot-bot
```

注意：

- 单跑 `executor` 不会自动起 `db`；如果业务需要 RPC 调 `DatabaseManager`，要保证它已经在跑（否则 RPC 失败）；
- 把 host/port 改到本机：`AGENTSERVER_HOST=localhost`，`DB_HOST=localhost` 等。

## 9. CLI 工具

```bash
poetry run cli --help

# 常用：
poetry run cli api-keys list --user-id ...
poetry run cli api-keys create --user-id ... --permissions execute_graph
poetry run cli credits grant --user-id ... --amount-credits 100
poetry run oauth-tool github authorize ...
poetry run export-api-schema       # 导出 OpenAPI JSON 给前端用
poetry run analytics-setup         # 创建分析视图
poetry run analytics-views         # 刷新视图
```

## 10. 跑测试

```bash
# 跑全部
poetry run test

# 单文件
poetry run pytest backend/api/features/store/db_test.py -xvs

# 单测试函数
poetry run pytest 'backend/blocks/test/test_block.py::test_available_blocks[GetCurrentTimeBlock]' -xvs
```

详见 [21 测试策略](./21-testing.md)。

## 11. 格式化与 Lint

```bash
poetry run format    # black + isort（会修复）
poetry run lint      # ruff（不会自动修，要手动）
```

提交前 `pre-commit` 钩子也会跑这些。

## 12. 调试技巧

### 12.1 让 ExecutionManager 单线程跑

设环境变量 `num_graph_workers=1`，断点能稳稳停在执行流里（多线程会让 step 跳来跳去）。

### 12.2 看 Prisma 真 SQL

```bash
PRISMA_LOG_QUERIES=true poetry run app
```

### 12.3 看 RabbitMQ 队列

```bash
docker compose exec rabbitmq rabbitmqctl list_queues name messages messages_ready
```

### 12.4 直连 PG

```bash
docker compose exec db psql -U postgres
\dt platform.*
SELECT id, name, "isActive" FROM platform."AgentGraph" LIMIT 10;
```

### 12.5 直连 Redis

```bash
# 单点
docker compose exec redis-0 redis-cli -p 17000 KEYS '*' | head -20
# 集群
docker compose exec redis-0 redis-cli -p 17000 -c
```

### 12.6 给 OpenAPI 看路由

```
http://localhost:8006/docs
http://localhost:8006/openapi.json
```

## 13. 重置环境

```bash
# 停所有容器并清数据
docker compose down -v

# 清 prisma 生成物
rm -rf backend/__generated__ /tmp/prisma-*

# 重新来一遍
docker compose up -d
poetry run prisma migrate deploy
poetry run app
```

## 14. Windows / WSL2

仓库主分支验证主要在 Linux / macOS。Windows 上：

- 推荐 WSL2 + Docker Desktop；
- 直接 Windows 跑 Python 也可；
- 路径相关（cloud_storage / workspace）走的全部是 Posix 语义；
- shell 用 bash（参考仓库 CLAUDE.md 的环境约定）。

## 15. 常见问题

| 现象                                                    | 排查                                                                   |
| ------------------------------------------------------- | ---------------------------------------------------------------------- |
| `prisma generate` 报 `unable to find binary`           | `poetry env info` 确认 Python 路径；删 `~/.cache/prisma-python` 重试   |
| `redis-cli cluster info` 不是 `cluster_state:ok`       | 重新 `docker compose down -v && up`，或手动跑 `redis-init`             |
| 502 from rest_server                                    | 看是不是 `migrate` 容器没成功完成；确认 `db` 健康                      |
| WebSocket 立刻 4002 关闭                                | `JWT_VERIFY_KEY` 与签发方不一致；或 token 过期                          |
| Block 执行扣费失败                                      | `ENABLE_CREDIT=true` 时余额不够 → grant credits；或检查 transactionKey |
| ClamAV 扫描超时                                         | 病毒库未下完；等几分钟或 `docker compose restart clamav`               |
| OAuth 回调 `redirect_uri_mismatch`                       | provider 控制台允许的 callback 必须与 `FRONTEND_BASE_URL` 一致         |
