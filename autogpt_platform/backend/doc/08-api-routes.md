# 08 API 路由清单

> 仅列出每个子模块的代表性核心端点，**不是全集**（路由总数 200+，全部以 OpenAPI 为准）。
> 完整 OpenAPI JSON 可由 `poetry run export-api-schema` 导出。

## 1. 主 API（前缀 `/api`，需 JWT）

### 1.1 Auth & 用户

| Method | URL                          | 描述                                          |
| ------ | ---------------------------- | --------------------------------------------- |
| POST   | `/auth/user`                 | JWT 验证后获取/创建 User 记录                 |
| POST   | `/auth/user/email`           | 更新邮箱                                      |
| GET    | `/auth/user/timezone`        | 获取/更新时区                                 |
| POST   | `/auth/user/timezone`        | 设置时区                                      |
| GET    | `/auth/api-keys`             | 列出当前用户的 API Key                        |
| POST   | `/auth/api-keys`             | 创建 API Key（指定 permissions）              |
| DELETE | `/auth/api-keys/{id}`        | 撤销 API Key                                  |

### 1.2 Onboarding

| Method | URL                                      | 描述                          |
| ------ | ---------------------------------------- | ----------------------------- |
| GET    | `/onboarding`                            | 获取引导进度                  |
| POST   | `/onboarding/step/{step}`                | 完成某一步                    |
| GET    | `/onboarding/recommended-agents`         | 推荐的 agent（来自 store）    |

### 1.3 Graphs

| Method | URL                                              | 描述                                      |
| ------ | ------------------------------------------------ | ----------------------------------------- |
| GET    | `/graphs`                                        | 列出我的 graph                            |
| POST   | `/graphs`                                        | 创建新 graph（首版本=1）                  |
| GET    | `/graphs/{graph_id}`                             | 取活跃版本详情                            |
| GET    | `/graphs/{graph_id}/versions/{version}`          | 取指定版本                                |
| PUT    | `/graphs/{graph_id}`                             | 创建新 version（保留历史）                |
| DELETE | `/graphs/{graph_id}`                             | 软删除                                    |
| PUT    | `/graphs/{graph_id}/versions/active`             | 切换活跃版本                              |
| POST   | `/graphs/{graph_id}/execute/{version}`           | 触发一次执行（→ RabbitMQ）                |

### 1.4 Executions

| Method | URL                                                         | 描述                                |
| ------ | ----------------------------------------------------------- | ----------------------------------- |
| GET    | `/executions`                                               | 我的执行历史（分页）                |
| GET    | `/executions/{exec_id}`                                     | 单次执行详情                        |
| POST   | `/executions/{exec_id}/cancel`                              | 取消（→ RabbitMQ cancel queue）     |
| GET    | `/executions/pending-reviews`                               | 列出所有待 HITL 审查的节点          |
| GET    | `/executions/{exec_id}/pending-reviews`                     | 当前执行的待审查项                  |
| POST   | `/executions/action`                                        | 批准/拒绝/编辑某个 HITL node        |

### 1.5 Blocks（前端 Builder 用）

| Method | URL                          | 描述                            |
| ------ | ---------------------------- | ------------------------------- |
| GET    | `/blocks`                    | 列出全部已注册 Block            |
| GET    | `/blocks/{block_id}`         | 单 Block 详细 schema            |
| POST   | `/blocks/{block_id}/execute` | 单 Block 直接执行（试运行）     |

### 1.6 Builder（搜索 / 推荐）

| Method | URL                                         | 描述                              |
| ------ | ------------------------------------------- | --------------------------------- |
| GET    | `/builder/blocks/suggestions`               | 根据上下文推荐 Block              |
| GET    | `/builder/blocks/categories`                | Block 分类                        |
| GET    | `/builder/search`                           | 搜 Block + Provider               |

### 1.7 Library

| Method | URL                                              | 描述                              |
| ------ | ------------------------------------------------ | --------------------------------- |
| GET    | `/library/agents`                                | 我的 agent 库                     |
| POST   | `/library/agents`                                | 加入 / 收藏                       |
| POST   | `/library/agents/{id}/fork`                      | fork 到新 graph                   |
| GET    | `/library/folders`                               | 文件夹列表                        |
| POST   | `/library/folders`                               | 新建文件夹                        |
| PATCH  | `/library/folders/{id}`                          | 改名 / 移动                       |
| POST   | `/library/folders/bulk-move-agents`              | 批量移动                          |
| GET    | `/library/presets`                               | 我的 preset                       |
| POST   | `/library/presets`                               | 创建 preset                       |
| POST   | `/library/presets/{id}/setup-trigger`            | 绑定 webhook 触发                 |
| POST   | `/library/presets/execute`                       | 用 preset 执行                    |

### 1.8 Store / Marketplace

| Method | URL                                            | 描述                              |
| ------ | ---------------------------------------------- | --------------------------------- |
| GET    | `/store/agents`                                | 列表（featured / sort / search）  |
| GET    | `/store/agents/{username}/{name}`              | 详情（带缓存）                    |
| GET    | `/store/creators`                              | 创作者列表                        |
| GET    | `/store/profile`                               | 我的创作者主页                    |
| POST   | `/store/profile`                               | 更新主页                          |
| GET    | `/store/submissions`                           | 我的发布记录                      |
| POST   | `/store/submissions`                           | 新建发布版本                      |
| PUT    | `/store/submissions/{id}`                      | 更新（含审核状态）                |
| DELETE | `/store/submissions/{id}`                      | 撤回                              |
| POST   | `/store/submissions/media/upload`              | 上传封面图                        |
| GET    | `/store/search`                                | hybrid search（语义 + BM25）      |
| POST   | `/store/agents/{id}/reviews`                   | 提交评论                          |

### 1.9 Integrations / OAuth

| Method | URL                                                   | 描述                              |
| ------ | ----------------------------------------------------- | --------------------------------- |
| GET    | `/integrations/providers`                             | 支持的 provider 列表              |
| POST   | `/integrations/oauth/initiate`                        | 启动 OAuth 流程                   |
| GET    | `/integrations/oauth/callback`                        | OAuth 回调                        |
| GET    | `/integrations/credentials`                           | 我的凭证                          |
| POST   | `/integrations/credentials`                           | 手动新增（API Key 类）            |
| DELETE | `/integrations/credentials/{id}`                      | 撤销                              |
| POST   | `/integrations/webhook/register`                      | 显式注册 webhook                  |

### 1.10 Credits / Billing

| Method | URL                                  | 描述                                  |
| ------ | ------------------------------------ | ------------------------------------- |
| GET    | `/credits`                           | 当前余额                              |
| GET    | `/credits/transactions`              | 交易流水                              |
| POST   | `/credits/top-up`                    | Stripe 充值                           |
| POST   | `/credits/subscribe`                 | 订阅                                  |
| POST   | `/credits/stripe_webhook`            | **公开** Stripe Webhook（无 JWT）     |
| POST   | `/credits/refund`                    | 退款申请                              |

### 1.11 Files / Workspace

| Method | URL                            | 描述                            |
| ------ | ------------------------------ | ------------------------------- |
| POST   | `/files/upload`                | 上传（含 ClamAV 扫描）          |
| GET    | `/files/{file_id}`             | 下载                            |
| GET    | `/workspace/files`             | 我的工作区文件                  |
| DELETE | `/workspace/files/{id}`        | 删除（软）                      |

### 1.12 Notifications & Push

| Method | URL                                 | 描述                              |
| ------ | ----------------------------------- | --------------------------------- |
| GET    | `/auth/notification-preferences`    | 通知开关                          |
| PUT    | `/auth/notification-preferences`    | 修改                              |
| POST   | `/push/subscribe`                   | 注册 Web Push 订阅                |
| DELETE | `/push/subscribe/{id}`              | 取消                              |
| GET    | `/push/vapid-key`                   | 拿 VAPID 公钥                     |

### 1.13 Chat / Copilot

| Method | URL                              | 描述                                          |
| ------ | -------------------------------- | --------------------------------------------- |
| GET    | `/chat/sessions`                 | 我的会话列表                                  |
| POST   | `/chat/sessions`                 | 新建会话                                      |
| POST   | `/chat/sessions/{id}/messages`   | 发送消息（→ RabbitMQ copilot_execution）      |
| GET    | `/chat/sessions/{id}/stream`     | SSE 流式获取助手回复                          |
| DELETE | `/chat/sessions/{id}`            | 删除                                          |

### 1.14 MCP（Model Context Protocol 暴露）

| Method | URL                          | 描述                                            |
| ------ | ---------------------------- | ----------------------------------------------- |
| GET    | `/mcp/server`                | 当前用户可暴露的 agent 作为 MCP tools 的清单     |
| POST   | `/mcp/invoke/{tool}`         | 通过 MCP 协议调用                                |

### 1.15 Otto（自然语言入口 / 内部）

| Method | URL                          | 描述                                            |
| ------ | ---------------------------- | ----------------------------------------------- |
| POST   | `/otto/...`                  | 内部 LLM 辅助路由（按需查源码）                 |

### 1.16 Admin（需 admin JWT）

| Method | URL                                         | 描述                            |
| ------ | ------------------------------------------- | ------------------------------- |
| GET    | `/admin/users`                              | 用户列表                        |
| POST   | `/admin/credits/grant`                      | 赠送积分                        |
| GET    | `/admin/diagnostics`                        | 系统健康                        |
| POST   | `/admin/store/listings/{id}/approve`        | 审核通过                        |
| POST   | `/admin/store/listings/{id}/reject`         | 拒绝                            |
| POST   | `/admin/cost/adjust`                        | 平台成本人工修正                |

## 2. External v1 API（前缀 `/external-api/v1`，API Key）

### 2.1 Identity / Blocks

| Method | URL                              | 描述                            |
| ------ | -------------------------------- | ------------------------------- |
| GET    | `/me`                            | 当前 API Key 对应的用户信息     |
| GET    | `/blocks`                        | 全部 Block 元数据               |
| POST   | `/blocks/{block_id}/execute`     | 单 Block 执行                   |

### 2.2 Graphs

| Method | URL                                                              | 描述                          |
| ------ | ---------------------------------------------------------------- | ----------------------------- |
| POST   | `/graphs`                                                        | 创建 graph                    |
| POST   | `/graphs/{graph_id}/execute/{version}`                           | 执行                          |
| GET    | `/graphs/{graph_id}/executions/{exec_id}/results`                | 取结果                        |

### 2.3 Integrations（OAuth via API Key）

| Method | URL                                  | 描述                              |
| ------ | ------------------------------------ | --------------------------------- |
| POST   | `/integrations/oauth/initiate`       | 启动 OAuth                        |
| POST   | `/integrations/oauth/complete`       | 提交 code                         |
| POST   | `/integrations/credentials/api-key`  | 上传 API Key 类凭证               |
| GET    | `/integrations/credentials`          | 列出                              |

### 2.4 Tools（无状态调用）

| Method | URL                  | 描述                                          |
| ------ | -------------------- | --------------------------------------------- |
| POST   | `/tools/find-agent`  | 按描述搜 agent                                |
| POST   | `/tools/run-agent`   | 立即运行或调度运行某个 agent                  |

### 2.5 Store（公开）

| Method | URL                | 描述                              |
| ------ | ------------------ | --------------------------------- |
| GET    | `/store/agents`    | 列表                              |

> External API 鉴权：HTTP `X-API-Key` 或 OAuth Bearer。每个端点用 `Security(require_permission(APIKeyPermission.XXX))` 校验权限。

## 3. WebSocket

详见 [09 WebSocket API](./09-api-websocket.md)。

| URL                              | 描述                                 |
| -------------------------------- | ------------------------------------ |
| `/ws/{graph_exec_id}?token=JWT`  | 订阅指定执行的事件流                 |
| `/ws/notifications?token=JWT`    | 用户级通知流                         |

## 4. 公开端点（无认证）

- `GET /health` —— 存活检查
- `GET /metrics` —— Prometheus
- `GET /docs`、`/redoc`、`/openapi.json`
- `POST /api/credits/stripe_webhook` —— Stripe（按签名校验）
- `POST /webhooks/{webhook_id}` —— 第三方 webhook ingress（按 provider 签名校验）
- `GET /store/...`（部分公开商品页）

## 5. 路径前缀对照

| 前缀                  | App                  | 鉴权        |
| --------------------- | -------------------- | ----------- |
| `/api/...`            | 主 AgentServer       | JWT         |
| `/external-api/v1/...`| External API 子应用  | API Key     |
| `/ws/...`             | WebsocketServer 进程 | JWT (query) |
| `/webhooks/...`       | 主 AgentServer       | provider 签名 |
| `/health`、`/metrics`、`/docs` | 主 AgentServer | 无          |

## 6. 端点新增清单

1. 在 `backend/api/features/<domain>/` 下加路由文件，定义 router 与 Pydantic 模型；
2. 鉴权：`Security(get_user_id)` / `Security(requires_admin_user)`；
3. 业务调用 `backend/data/...`，**不要**直接 import Prisma；
4. 错误用 `backend/util/exceptions.py` 抛；
5. 加 colocated `*_test.py`，含 snapshot；
6. 在 `backend/api/features/v1.py` `include_router(...)`；
7. 跑 `poetry run test` 与 `poetry run lint`；
8. 如果端点要被前端用：`pnpm generate:api`（前端项目）。
