# 07 REST API 总览

## 1. 入口

主 REST 服务 = [`backend/api/rest_api.py`](../backend/api/rest_api.py) 中的 `AgentServer(AppProcess)`。

```python
class AgentServer(AppProcess):
    service_name = "AgentServer"
    def run(self):
        uvicorn.run(
            "backend.api.rest_api:app",
            host=settings.AGENTSERVER_HOST,
            port=settings.AGENTSERVER_PORT,    # 默认 8006
            ...
        )
```

`backend.api.rest_api:app` 是一个 FastAPI 实例，挂载顺序：

1. `lifespan` 上下文（启动钩子）；
2. CORS 中间件 + GZip 中间件 + `SecurityHeadersMiddleware`；
3. Prometheus instrumentor 暴露 `/metrics`；
4. `app.mount("/external-api", external_api)` 挂载对外 v1 API 子应用；
5. `include_router(v1_router)` —— 把 `backend/api/features/v1.py` 聚合的总路由装上。

## 2. lifespan 阶段做了什么

```python
async def lifespan(app):
    await prisma.connect()                     # PG 连接池
    await redis_client.connect_eager()         # 失败则进程立刻退出（fail fast）
    init_supabase_client()                     # 验证 SUPABASE_URL/KEY
    feature_flag.initialize()                  # 拉取 LaunchDarkly
    initialize_blocks()                        # 同步 AgentBlock 元数据
    yield
    await disconnect_all()
```

## 3. 中间件栈

### 3.1 SecurityHeadersMiddleware

[`backend/api/middleware/security.py`](../backend/api/middleware/security.py)

- **默认禁用一切缓存**：在响应上覆盖
  ```
  Cache-Control: no-store, no-cache, must-revalidate, private
  Pragma: no-cache
  ```
- **白名单（CACHEABLE_PATHS）**：静态资源、`/static/*`、`/_next/static/*`、健康检查、Store 公开页、文档；
- 安全响应头：
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
  - `X-XSS-Protection: 1; mode=block`
  - `Referrer-Policy: strict-origin-when-cross-origin`

> 加新可缓存的端点 → 改 `CACHEABLE_PATHS`，并 review 是否泄漏 PII。

### 3.2 CORS

`backend/api/utils/cors.py` 根据 settings 动态配置允许的 origin / 方法 / 头；默认开放 `Authorization`、`Content-Type`。

### 3.3 GZip

阈值通常 1 KiB 以上自动压缩；不压缩 SSE / WebSocket 路径。

### 3.4 Prometheus

`prometheus-fastapi-instrumentator` 自动统计每个路由的 latency / count / status；指标名前缀 `http_`。

## 4. 路由聚合

[`backend/api/features/v1.py`](../backend/api/features/v1.py) 是聚合点，把每个 feature 的子 router include 进去。父 router 默认依赖 `requires_user`，子 router 可以覆盖（如 admin）。

简化结构：

```python
v1_router = APIRouter(prefix="/api", dependencies=[Security(get_user_id)])

v1_router.include_router(graphs.router,        prefix="/graphs")
v1_router.include_router(blocks.router,        prefix="/blocks")
v1_router.include_router(credits.router,       prefix="/credits")
v1_router.include_router(integrations.router,  prefix="/integrations")
v1_router.include_router(library.router,       prefix="/library")
v1_router.include_router(store.router,         prefix="/store")
v1_router.include_router(executions.router,    prefix="/executions")
v1_router.include_router(builder.router,       prefix="/builder")
v1_router.include_router(chat.router,          prefix="/chat")
v1_router.include_router(workspace.router,     prefix="/workspace")
v1_router.include_router(onboarding.router,    prefix="/onboarding")
v1_router.include_router(push.router,          prefix="/push")
v1_router.include_router(otto.router,          prefix="/otto")
v1_router.include_router(mcp.router,           prefix="/mcp")
v1_router.include_router(admin.router,         prefix="/admin")
...
```

详细端点速查见 [08 API 路由清单](./08-api-routes.md)。

## 5. 鉴权与依赖注入

### 5.1 JWT 解析

`autogpt_libs.auth.jwt_utils.get_jwt_payload(authorization)`：从 `Authorization: Bearer <token>` 头解析、验签（`JWT_VERIFY_KEY`）、返回 payload。

依赖封装：

| 依赖                    | 含义                                                       |
| ----------------------- | ---------------------------------------------------------- |
| `requires_user`         | 必须存在已登录用户                                         |
| `requires_admin_user`   | 必须有 admin 角色                                          |
| `get_user_id`           | 直接拿 `user_id`（推荐用于业务逻辑）                       |
| `get_user`              | 拿到完整 `User`（多走一次 DB 查询）                        |

> **务必使用 `Security()` 而非 `Depends()` 来注入鉴权依赖** —— 后者不会出现在 OpenAPI 的 `security` 段。

```python
@router.post("/{graph_id}/execute/{version}")
async def execute_graph(
    graph_id: str,
    version: int,
    payload: ExecuteGraphRequest,
    user_id: Annotated[str, Security(get_user_id)],
):
    ...
```

### 5.2 鉴权可关

`ENABLE_AUTH=false` 时，所有依赖直接返回 `DEFAULT_USER_ID`，便于本地调试。**不要在 staging / prod 关。**

## 6. Pydantic 共享模型

[`backend/api/model.py`](../backend/api/model.py) 定义跨路由复用的 schema：

- `WSMessage` / `WSMethod` —— WebSocket 协议；
- `CreateGraph` / `SetGraphActiveVersion` —— Graph 操作；
- `CreateAPIKeyRequest` / `UpdatePermissionsRequest` —— API Key；
- `UploadFileResponse`、`TimezoneResponse` —— 通用响应。

## 7. 错误处理与 Sentry

- FastAPI 默认抛 `HTTPException` 由框架转 JSON；
- 业务异常统一在 `backend/util/exceptions.py`：`InsufficientBalanceError`、`PermissionDeniedError`、`NotFoundError`、`ValidationError` 等；
- 全局异常 → Sentry（已通过 `sentry-sdk[fastapi]` 集成自动捕获）。
- 错误消息中**不要直接拼绝对路径** —— 用 `os.path.basename()`。

## 8. 文件上传与 store_media_file

`POST /api/files/upload` 走 `python-multipart`：
1. 接收 `UploadFile`；
2. 调 `store_media_file()`（[`backend/util/file.py`](../backend/util/file.py)），它负责：
   - 病毒扫描（ClamAV）；
   - 持久化到 GCS / 本地；
   - 返回三种格式之一（`for_local_processing` / `for_external_api` / `for_block_output`）。

## 9. SSE 路径（Copilot Chat）

Copilot chat 端点（`/api/chat/...`）输出 SSE：
- `data:` 行 → 前端 Zod schema 严格匹配；
- `: comment` 行 → 心跳；
- 不能被 GZip 或缓存中间件破坏 —— 路径在白名单。

## 10. 限流

- Copilot 路径：`backend/copilot/rate_limit.py` 基于 Redis 分布式计数器；
- 普通 API 路径：暂未默认全局限流，依赖订阅层（subscriptionTier）做配额；
- 充值/支付 webhook 不能被限流。

## 11. 启动与端口

```bash
# 单进程方式
poetry run rest

# 容器（docker-compose.platform.yml: rest_server）
command: ["rest"]
ports:   ["8006:8006"]
```

环境变量：
- `AGENTSERVER_HOST` / `AGENTSERVER_PORT`
- `FRONTEND_BASE_URL` —— 用于 CORS / 邮件链接 / OAuth 重定向
- `PLATFORM_BASE_URL` —— 给第三方 webhook 拼回调 URL 用

## 12. 调用流示意

```
Client ── HTTPS ──► Nginx/LB ──► AgentServer
                                  ├─ Middleware: Security/CORS/GZip
                                  ├─ Auth: JWT → user_id
                                  ├─ Route handler
                                  │   ├─ Read: data layer (Prisma/Redis)
                                  │   ├─ Write: data layer
                                  │   └─ Publish: rabbitmq / event_bus
                                  └─ Response (Pydantic → JSON)
```
