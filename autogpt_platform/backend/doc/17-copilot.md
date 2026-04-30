# 17 Copilot

## 1. 角色

**Copilot** 是平台自带的 AI 助手 / Agent，覆盖三类场景：

1. **Builder Copilot**：在前端 Builder 里帮用户解释 Block、推荐图结构、自动接线；
2. **Chat Copilot**：用户在网页 chat 面板里聊天，可调用 Block 作为 tool；
3. **Bot Bridge**：把 Copilot 接到 Discord / Telegram，等同于"在群里 @bot"。

源码：[`backend/copilot/`](../backend/copilot/)，是后端最大的子模块之一。

## 2. 进程拆分

| 进程                  | 类                                          | 职责                                                  |
| --------------------- | ------------------------------------------- | ----------------------------------------------------- |
| AgentServer           | `backend/api/features/chat/`                | 接受 HTTP 请求 / SSE 流式返回                         |
| **CoPilotExecutor**   | `backend/copilot/executor/manager.py`       | 消费 RabbitMQ `copilot_execution_queue`，跑实际推理   |
| **CoPilotChatBridge** | `backend/copilot/bot/app.py`                | Discord / Telegram bot 适配器、对外 RPC               |
| PlatformLinkingManager| `backend/platform_linking/manager.py`       | 平台账号绑定（Copilot 知道你是哪个用户）              |

把 Copilot 的执行从主 ExecutionManager 拆出去，是因为 Copilot 是长任务（流式输出几十秒），不能挤占图执行 worker。

## 3. 双执行链

`backend/copilot/` 下并存三套实现，按场景路由：

| 路径                    | 文件                                                  | 用途                                              |
| ----------------------- | ----------------------------------------------------- | ------------------------------------------------- |
| **SDK 路径**            | `backend/copilot/sdk/`                                | 基于 [Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python)，支持 extended thinking、MCP 工具、子 agent |
| **Baseline 路径**       | `backend/copilot/baseline/`                           | 传统 OpenAI / Anthropic completion 流程，工具调用循环手写 |
| **Dummy**               | `backend/copilot/baseline/dummy.py` 等                | 测试 / 不付费时的兜底                             |

### 3.1 model_router

[`backend/copilot/model_router.py`](../backend/copilot/model_router.py)

按 `ChatConfig`（`copilot/config.py`）的层级（`standard / advanced`）选 provider：

- `standard` → OpenAI / Groq / Ollama；
- `advanced` → Anthropic（带 thinking）；
- 配额不足或 disabled → Dummy。

### 3.2 SDK 特性

`copilot/sdk/`：

- `service.py` —— 流式处理；
- `compaction.py` —— 长对话历史 LLM 摘要压缩；
- `env.py` —— SDK 进程环境变量；
- 自带 prompt cache（与 Anthropic 5min TTL 配合）；
- 兼容 MCP（用户暴露的 agent 即 MCP tools）。

### 3.3 Baseline 特性

`copilot/baseline/`：

- `service.py` —— 标准 chat completion；
- `reasoning.py` —— 可选推理步骤；
- 工具调用循环：`backend/util/tool_call_loop.py`。

## 4. 核心组件

### 4.1 Service 层

[`backend/copilot/service.py`](../backend/copilot/service.py)

- 构建系统提示（融合 `BusinessUnderstanding`、当前 user 信息）；
- 集成 [Langfuse](https://langfuse.com)：系统提示外置 + 调用观测；
- 写 token 计费记录；
- 生成会话标题（首条消息后用 LLM 生成）。

### 4.2 上下文与工具

| 文件                                            | 说明                                                          |
| ----------------------------------------------- | ------------------------------------------------------------- |
| `context.py`                                    | `CopilotContext`：单次 turn 的上下文（user、session、in-flight credentials） |
| `builder_context.py`                            | Builder Copilot 专用上下文（当前打开的 graph、选中的 node）     |
| `tools/`                                        | 可暴露给 LLM 调用的工具实现（搜 Block、检索 Library、查执行历史…） |
| `pending_messages.py`                           | 用户消息排队 / 合并（速率限制后过滤）                         |
| `pending_message_helpers.py`                    | helper                                                        |

### 4.3 Processor

[`backend/copilot/executor/manager.py`](../backend/copilot/executor/manager.py)

- 单 worker 池消费 `copilot_execution_queue`；
- 每个 turn 跑一次 `CoPilotProcessor.process()`；
- 双锁：refresh lock（凭证刷新合并）+ acquire lock（防 token 替换）；
- 异步取消：用户在前端关掉会话 → 优雅 cancel；
- Sync fail-close：Redis CAS 失败时回退到本地排队，保证消息不丢。

### 4.4 Stream Registry

[`backend/copilot/stream_registry.py`](../backend/copilot/stream_registry.py)

- Redis Streams + Sharded PubSub 实现的"流式输出 buffer"；
- 客户端 SSE 接 `GET /api/chat/sessions/{id}/stream`；
- 即使页面刷新，可以用 `last_event_id` 续传未读 chunk。

### 4.5 Thinking Stripper

[`backend/copilot/thinking_stripper.py`](../backend/copilot/thinking_stripper.py)

- 把 Anthropic 的 `thinking` 段过滤掉再发给前端（避免泄漏 + 节省带宽）；
- 在出错复现时仍可保留 thinking 给 admin 调试。

### 4.6 Token Tracking

[`backend/copilot/token_tracking.py`](../backend/copilot/token_tracking.py)

- 每次 LLM 响应记录 input / output / reasoning / cache token；
- 写 `ChatSession.totalPromptTokens / totalCompletionTokens`；
- 同时调 `log_platform_cost()` 与 `spend_credits()`（参见 [16](./16-credit-billing.md)）。

### 4.7 Rate Limit

[`backend/copilot/rate_limit.py`](../backend/copilot/rate_limit.py)

- 用户级 + IP 级；
- Redis 分布式计数器，滑窗实现；
- 倍率 = 全局 limit × `User.subscriptionTier` 的倍率（BASIC=1×, PRO=5×, MAX=20×, BUSINESS=60×, ENTERPRISE=60×）；
- 超限 → 429 + retry-after。

### 4.8 Permissions

[`backend/copilot/permissions.py`](../backend/copilot/permissions.py)

- 对 Copilot 调用的"图执行权限"做校验（不同于一般 API 端点的权限模型）；
- 防止用户绕过限制：例如用 Copilot 调一个非自己的 graph。

## 5. Bot 桥接

### 5.1 适配器

[`backend/copilot/bot/`](../backend/copilot/bot/)：

- `app.py` —— 进程入口 `CoPilotChatBridge`；
- `adapters/discord/adapter.py` —— Discord bot；
- `handler.py` —— 命令解析 + 消息路由；
- `threads.py` —— Discord 线程会话，TTL 7 天。

### 5.2 流程

```
Discord user @bot "..."
  └─ DiscordAdapter.on_message
       └─ PlatformLinkingManager.resolve_user_link("discord", user_id)
              → AutoGPT user_id
       └─ pending_messages.enqueue(user_id, content)
       └─ rabbitmq.publish(copilot_execution_queue, ...)

CoPilotExecutor 处理 → stream_registry 写流
  └─ CoPilotChatBridge subscribes 到流 → 发回 Discord channel（chunked）
```

## 6. 与平台数据耦合

| 表                          | 用途                                                |
| --------------------------- | --------------------------------------------------- |
| `ChatSession`               | 一次会话；包含 metadata、credentials snapshot       |
| `CoPilotUnderstanding`      | 用户级"业务理解"，注入系统提示                       |
| `BuilderSearchHistory`      | Builder Copilot 搜索的 Block 历史                    |
| `User.subscriptionTier`     | 限流倍率                                             |

## 7. 数据访问

[`backend/copilot/db.py`](../backend/copilot/db.py)：封装 ChatSession / message / 用户偏好的 CRUD。

## 8. Graphiti 时序记忆（可选）

[`backend/copilot/graphiti/`](../backend/copilot/graphiti/)

- 基于 [graphiti-core](https://github.com/getzep/graphiti) + FalkorDB；
- 用户与 Copilot 的对话事实抽取后入图；
- 后续 turn 检索相关 fact 拼到上下文；
- 由 LaunchDarkly Flag `graphiti-memory` 控制启停；
- 配置：`GRAPHITI_FALKORDB_HOST/PORT/PASSWORD`、`GRAPHITI_LLM_MODEL`、`GRAPHITI_EMBEDDER_MODEL`。

## 9. 与 Block 的对应

Copilot 不复用 ExecutionManager 的执行链路（Copilot 内部直接调 LLM SDK），但**调用 Block 时**会反过来：

- Copilot 把 Block 包装成 LLM tool；
- LLM 决定调哪个，然后 Copilot 在自己进程内调 `block.execute(...)`；
- 不走 RabbitMQ execution_queue（避免 round-trip）。

## 10. SSE 协议

`GET /api/chat/sessions/{id}/stream` 返回：

```
: keep-alive heartbeat

data: {"type":"thinking_start"}
data: {"type":"text","delta":"Hello "}
data: {"type":"text","delta":"world"}
data: {"type":"tool_use","name":"FindAgent","args":{...}}
data: {"type":"tool_result","content":[...]}
data: {"type":"text","delta":"."}
data: {"type":"done"}
```

前端用 Zod schema 严格校验，未识别字段 ignore。任何 GZip / 缓存中间件都要绕过这条路径（`SecurityHeadersMiddleware` 已加白名单）。

## 11. 关键测试

- `service_test.py` / `service_e2e_test.py` —— 端到端模拟 turn；
- `model_router_test.py` —— 路由策略；
- `rate_limit_test.py` —— 限流；
- `prompt_cache_test.py` —— Anthropic prompt cache 行为；
- `permissions_test.py` —— 越权防护；
- `pending_messages_test.py` / `stream_registry_test.py` —— 队列与流；
- `transcript_*_test.py` —— transcript 渲染。

## 12. 进一步阅读

- 仓库 `docs/platform/` 下的 Copilot 设计文档（如有）；
- Anthropic Prompt Caching 文档（影响 SDK 路径配置）；
- Langfuse 后台 → 看实际 trace。
