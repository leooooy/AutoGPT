# 02 技术栈

## 1. 运行时

| 组件             | 版本                       | 备注                                                                            |
| ---------------- | -------------------------- | ------------------------------------------------------------------------------- |
| Python           | `>=3.10, <3.14`            | `pyproject.toml:11`，CI 推荐 3.12                                               |
| Poetry           | 2.2.1（pinned）            | 依赖与脚本管理；升级前必须确认 dependabot 兼容                                  |
| Uvicorn          | ^0.40.0                    | FastAPI 默认 ASGI server，启用 `extras=["standard"]`                            |
| Pyro5（间接）    | via `autogpt_libs`         | 进程间 RPC 框架，被 `AppService` 封装                                           |

## 2. Web / API 框架

| 库                                | 用途                                          |
| --------------------------------- | --------------------------------------------- |
| `fastapi ^0.128.6`                | REST 与 WebSocket 框架                        |
| `pydantic[email] ^2.12.5`         | Schema 校验、序列化                           |
| `pydantic-settings ^2.12.0`       | 基于环境变量的配置（`backend/util/settings.py`） |
| `prometheus-fastapi-instrumentator` | 自动暴露 `/metrics`                          |
| `python-multipart`                | 文件上传                                     |
| `bleach[css]`                     | 富文本/HTML 清洗（Store profile 等）         |

## 3. 数据 / 存储

| 库 / 服务                 | 用途                                                          |
| ------------------------- | ------------------------------------------------------------- |
| `prisma ^0.15.0`          | PostgreSQL ORM，schema 在 `backend/schema.prisma`              |
| `psycopg2-binary`         | 同步 PG 驱动（迁移、APScheduler 内部使用）                    |
| `sqlalchemy ^2.0`         | APScheduler 持久化、原生统计查询                              |
| `redis ^6.2`              | Redis Cluster 客户端（同步 + 异步）                           |
| `pika ^1.3` / `aio-pika`  | RabbitMQ（同步发送 + 异步消费）                               |
| `pgvector` 扩展           | 向量索引（Store hybrid search、Copilot embeddings）           |
| `falkordb ^1.1`           | Copilot Graphiti 用的图数据库（共享 Redis 协议）              |
| `gcloud-aio-storage` / `google-cloud-storage` | GCS 文件存储                              |

## 4. 鉴权与安全

| 库 / 服务                | 用途                                                          |
| ------------------------ | ------------------------------------------------------------- |
| `supabase 2.28.0`        | 第三方鉴权（用户邮箱/密码、OAuth Google/GitHub 等）           |
| `cryptography ^46`       | 凭证 AES 加密（`backend/util/encryption.py`）                  |
| `pywebpush ^2.3`         | Web Push（VAPID）                                              |
| `aioclamd ^1.0`          | ClamAV 异步病毒扫描                                           |
| `autogpt_libs.auth`      | 仓库内库；JWT 解析、`requires_user` / `requires_admin_user`    |

## 5. 异步与并发

| 库                | 用途                                                         |
| ----------------- | ------------------------------------------------------------ |
| `apscheduler ^3.11`| 定时任务（Cron / OneShot）                                  |
| `tenacity ^9.1`   | 通用重试装饰器（`backend/util/retry.py`）                    |
| `cachetools`      | LRU + TTL 缓存（用户、block 元数据等）                        |
| `aiohttp / aiodns`| 异步 HTTP                                                    |
| `httpx`           | 测试用同步/异步双模 HTTP                                     |

## 6. AI / LLM

| 库                       | 用途                                                                |
| ------------------------ | ------------------------------------------------------------------- |
| `anthropic ^0.79`        | Claude 调用（Copilot Baseline、LLM Block）                          |
| `openai ^1.97`           | OpenAI（含 Responses API）                                          |
| `groq ^0.30`             | Groq Cloud LLM                                                      |
| `ollama ^0.6`            | 本地模型                                                            |
| `claude-agent-sdk ^0.1.64`| Copilot SDK 路径                                                   |
| `mem0ai`                 | Mem0 长期记忆                                                       |
| `pinecone ^7.3`          | 向量数据库 Block                                                    |
| `tiktoken ^0.12`         | Token 计数                                                          |
| `langfuse ^3.14`         | Prompt 管理与观测（Copilot 系统提示外置）                           |
| `langsmith ^0.7`         | 选用观测                                                            |
| `replicate / fal / elevenlabs / e2b / e2b-code-interpreter` | 多媒体与代码沙箱       |
| `graphiti-core ^0.28`    | 时序知识图（FalkorDB 后端）                                         |

## 7. 第三方集成 SDK

只列代表，完整见 `pyproject.toml`：

`google-api-python-client`, `google-auth-oauthlib`, `googlemaps`, `discord-py`, `tweepy`, `praw` (Reddit),
`todoist-api-python`, `firecrawl-py`, `exa-py`, `feedparser`, `youtube-transcript-api`, `yt-dlp`,
`postmarker` (邮件), `stripe ^11.5`, `posthog`, `agentmail`, `stagehand`, `gravitas-md2gdocs`, `gravitasml`,
`fpdf2`, `openpyxl`, `pandas`, `pyarrow`, `pymssql`, `pymysql`, `sqlparse`, `bm25 (rank-bm25)`, `regex`, `croniter`, `jinja2`, `jsonschema`, `jsonref`, `html2text`.

## 8. 监控、日志、错误

| 库 / 服务                       | 用途                                                          |
| ------------------------------- | ------------------------------------------------------------- |
| `sentry-sdk` (extras: anthropic, fastapi, launchdarkly, openai, sqlalchemy) | 错误上报与性能采样 |
| `prometheus-client` + `prometheus-fastapi-instrumentator` | Prometheus 指标               |
| `launchdarkly-server-sdk ^9.14` | Feature flag                                                  |
| `posthog ^7.6`                  | 产品分析                                                      |
| `autogpt_libs.logging`          | 结构化日志，云日志开关                                        |

## 9. 开发与质量

| 工具                 | 用途                                                                  |
| -------------------- | --------------------------------------------------------------------- |
| `pytest` + `pytest-asyncio` + `pytest-mock` + `pytest-snapshot` | 测试栈                                |
| `pytest-cov` / `pytest-watcher` | 覆盖率与监听                                              |
| `ruff ^0.15`         | Lint                                                                 |
| `black ^24.10`       | 格式化                                                               |
| `isort ^5.13`        | Import 排序（profile=black）                                          |
| `pyright ^1.1.407`   | 类型检查                                                             |
| `pre-commit ^4.4`    | 提交前钩子                                                           |
| `poethepoet`         | Poetry 任务编排                                                       |
| `faker`              | 测试数据                                                             |

## 10. 重要 Poetry 脚本（`pyproject.toml:126`）

```bash
poetry run app                     # 一键启动所有进程（开发用）
poetry run rest                    # 仅 REST 服务
poetry run ws                      # 仅 WebSocket
poetry run db                      # DatabaseManager
poetry run executor                # ExecutionManager
poetry run scheduler               # Scheduler
poetry run notification            # NotificationManager
poetry run platform-linking-manager
poetry run copilot-executor
poetry run copilot-bot

poetry run cli ...                 # CLI 工具集
poetry run test                    # 跑测试（带 docker postgres + prisma）
poetry run format                  # black + isort
poetry run lint                    # ruff
poetry run gen-prisma-stub         # 生成 Prisma 类型 stub
poetry run export-api-schema       # 导出 OpenAPI JSON
poetry run oauth-tool ...          # OAuth 调试
```

## 11. 整体依赖图速览

```
FastAPI + Pydantic ──┬── Prisma ──── PostgreSQL (+ pgvector)
                     ├── redis ──── Redis Cluster (3 shards)
                     ├── pika / aio-pika ──── RabbitMQ
                     ├── supabase ──── Supabase Auth
                     ├── pywebpush ──── Web Push services
                     ├── cryptography ──── 凭证加密
                     ├── apscheduler ──── 定时任务
                     ├── anthropic / openai / groq / ollama ──── LLM
                     ├── claude-agent-sdk ──── Copilot SDK
                     ├── graphiti-core ──── FalkorDB
                     ├── google-cloud-storage ──── GCS
                     ├── stripe ──── 订阅 / 充值
                     └── 50+ 第三方 SDK ──── Block 集成
```
