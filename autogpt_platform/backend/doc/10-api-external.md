# 10 External API（v1）

## 1. 用途

External API 是一套面向**第三方应用 / SDK / Bot** 的对外 HTTP 接口，与主 REST API 是**两套独立鉴权、独立路由**：

| 维度        | 主 API (`/api/...`)         | External API (`/external-api/v1/...`) |
| ----------- | --------------------------- | ------------------------------------- |
| 鉴权        | Supabase JWT                | API Key / OAuth Bearer                |
| 调用者      | 浏览器前端                  | 第三方系统、AutoPilot SDK、CLI       |
| 中间件      | 共享主应用的 SecurityHeaders | 独立的 `external/middleware.py`       |
| OpenAPI tag | 跟 feature 走               | 全部 `external`                       |

主应用通过 `app.mount("/external-api", external_api)` 把这套子应用挂上去。

## 2. 入口文件

```
backend/api/external/
├── fastapi_app.py     # external_api = FastAPI(...)
├── middleware.py      # require_auth / require_permission
└── v1/
    ├── routes.py      # /me, /blocks, /graphs ...
    ├── integrations.py # /integrations/...
    └── tools.py       # /tools/find-agent, /tools/run-agent
```

`fastapi_app.py` 也使用 `SecurityHeadersMiddleware`，在 External API 上同样禁用缓存。

## 3. 鉴权模型

### 3.1 API Key

- 用户在 `/api/auth/api-keys` 创建；
- 数据库表 `APIKey`，存哈希；
- 调用时：`X-API-Key: <plain-text>` 头；
- 服务端 lookup → 校验 → 注入 `APIAuthorizationInfo` 到请求；
- **Permission 字段**：`permissions: list[APIKeyPermission]`，按枚举控制粒度：
  - `IDENTITY` —— 调 `/me`
  - `READ_BLOCK` —— 列 / 读 Block
  - `EXECUTE_BLOCK` —— 单 Block 执行
  - `READ_GRAPH` —— 读 graph
  - `WRITE_GRAPH` —— 写 / 创建 graph
  - `EXECUTE_GRAPH` —— 触发执行
  - `READ_EXECUTION` —— 读执行结果
  - `MANAGE_INTEGRATIONS` —— OAuth / 凭证操作
  - `READ_STORE` —— 商城读取
  - …其他

### 3.2 OAuth Bearer（平台作为 OP）

平台同时充当 OAuth Provider（见 [05 数据模型](./05-data-model.md) `OAuth*` 表组）：

- 第三方应用通过标准 Authorization Code + PKCE 拿 access_token；
- 访问 External API 时 `Authorization: Bearer <token>`；
- 同样进入 `APIAuthorizationInfo`，但 permission 来自 OAuth scope。

### 3.3 依赖封装

```python
auth: APIAuthorizationInfo = Security(
    require_permission(APIKeyPermission.WRITE_GRAPH)
)
```

`require_permission(*perms)` 内部：

1. `require_auth()` 读出 `APIAuthorizationInfo`（API Key 或 OAuth）；
2. 确认 `auth.permissions` 包含传入的全部 perm；
3. 不通过则 401 / 403。

> 始终用 `Security(...)`，让 OpenAPI 正确暴露 `securitySchemes`。

## 4. 主要端点

### 4.1 `routes.py`（Identity / Blocks / Graphs）

| Method | URL                                                              | Permission             |
| ------ | ---------------------------------------------------------------- | ---------------------- |
| GET    | `/me`                                                            | `IDENTITY`             |
| GET    | `/blocks`                                                        | `READ_BLOCK`           |
| GET    | `/blocks/{block_id}`                                             | `READ_BLOCK`           |
| POST   | `/blocks/{block_id}/execute`                                     | `EXECUTE_BLOCK`        |
| GET    | `/graphs`                                                        | `READ_GRAPH`           |
| POST   | `/graphs`                                                        | `WRITE_GRAPH`          |
| GET    | `/graphs/{graph_id}`                                             | `READ_GRAPH`           |
| POST   | `/graphs/{graph_id}/execute/{version}`                           | `EXECUTE_GRAPH`        |
| GET    | `/graphs/{graph_id}/executions/{exec_id}`                        | `READ_EXECUTION`       |
| GET    | `/graphs/{graph_id}/executions/{exec_id}/results`                | `READ_EXECUTION`       |

### 4.2 `integrations.py`

| Method | URL                                       | Permission              |
| ------ | ----------------------------------------- | ----------------------- |
| POST   | `/integrations/oauth/initiate`            | `MANAGE_INTEGRATIONS`   |
| POST   | `/integrations/oauth/complete`            | `MANAGE_INTEGRATIONS`   |
| POST   | `/integrations/credentials/api-key`       | `MANAGE_INTEGRATIONS`   |
| GET    | `/integrations/credentials`               | `MANAGE_INTEGRATIONS`   |
| DELETE | `/integrations/credentials/{id}`          | `MANAGE_INTEGRATIONS`   |

### 4.3 `tools.py`（无状态高层工具）

| Method | URL                  | 描述                                                                  |
| ------ | -------------------- | --------------------------------------------------------------------- |
| POST   | `/tools/find-agent`  | 自然语言搜 agent，返回候选列表（用于 SDK 让 LLM 选择）                |
| POST   | `/tools/run-agent`   | 立即触发或调度（cron）一个 agent，返回 `execution_id` 或 `schedule_id`|

### 4.4 Store 公开端点

| Method | URL                  | 说明                                  |
| ------ | -------------------- | ------------------------------------- |
| GET    | `/store/agents`      | 商城列表（部分字段对未鉴权请求也开放） |

## 5. 请求示例

```bash
# 获取我的信息
curl https://platform/external-api/v1/me \
  -H "X-API-Key: ak_live_xxx"

# 创建 graph
curl https://platform/external-api/v1/graphs \
  -H "X-API-Key: ak_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{"name": "demo", "nodes": [...], "links": [...]}'

# 触发执行
curl https://platform/external-api/v1/graphs/<id>/execute/1 \
  -H "X-API-Key: ak_live_xxx" \
  -d '{"inputs": {"query": "hello"}}'
```

## 6. 与第三方 webhook 接收的关系

第三方 → 平台的 webhook（如 GitHub push）走 `/webhooks/{webhook_id}`，**不属于** External API（无 API Key）—— 它的鉴权基于 provider 签名 + 已注册的 webhook secret。详见 [15 集成与 OAuth](./15-integrations-oauth.md)。

## 7. 速率限制 & 配额

- API Key 与 OAuth token 都按用户聚合 quota；
- 高级速率限制（Copilot 类）走 [`backend/copilot/rate_limit.py`](../backend/copilot/rate_limit.py) 的 Redis 计数器；
- 大批量调用建议错峰；执行类调用最终都进 RabbitMQ，限流主要在 *提交* 端。

## 8. 调试技巧

```bash
# 列出所有 External API 端点
curl https://platform/external-api/v1/openapi.json | jq '.paths | keys'

# 从 CLI 创建 / 撤销 API Key
poetry run cli api-keys list --user-id <id>
poetry run cli api-keys create --user-id <id> --permissions execute_graph,read_execution
```

## 9. 与主 API 的代码共享

External API 的 handler 内部仍调 `backend/data/...` 的同一套数据访问层；只是把 `user_id` 从 `APIAuthorizationInfo.user_id` 而不是 JWT 中拿。这意味着新增数据访问函数，主 API 与 External API 都自动可用。
