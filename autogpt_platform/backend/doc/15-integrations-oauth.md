# 15 集成、OAuth 与 Webhook

## 1. 总览

`backend/integrations/` 是平台与第三方服务的桥梁，负责：

1. **凭证（Credentials）**：用户的 OAuth Token / API Key 的存储、刷新、加锁、加密；
2. **OAuth Handler**：50+ 第三方 OAuth 流程的统一接口；
3. **Webhook Manager**：第三方推送进来的事件统一接收，转给 Block 触发；
4. **Managed Providers**：平台自家托管的 SaaS 凭证（如 Ayrshare），用户无感知。

```
backend/integrations/
├── credentials_store.py      # 加密读写（落 User.integrations 字段）
├── creds_manager.py          # IntegrationCredentialsManager（双锁 + 刷新）
├── managed_credentials.py    # 平台托管凭证
├── managed_providers/        # 自动 SSO（Ayrshare 等）
├── providers.py              # ProviderName 动态枚举
├── ayrshare.py               # 社交媒体聚合器
├── oauth/                    # 各 provider OAuth handler
│   ├── _base.py
│   ├── google.py / github.py / notion.py / twitter.py / ...
│   └── ...
└── webhooks/                 # Webhook ingress 与各 provider manager
    ├── _base.py
    ├── github.py / telegram.py / slack.py / ...
    └── utils.py
```

## 2. 三层凭证架构

```
┌────────────────────────────────────────┐
│ managed_credentials.py                 │ 自动注入（系统级）
└──────────────┬─────────────────────────┘
               │
┌──────────────▼─────────────────────────┐
│ creds_manager.py (IntegrationCredentialsManager)
│  ├─ acquire()     # 取凭证 + 加 acquire 锁
│  ├─ refresh()     # 刷新 token + 加 refresh 锁
│  └─ register_creds_changed_hook()     # 通知订阅者
└──────────────┬─────────────────────────┘
               │
┌──────────────▼─────────────────────────┐
│ credentials_store.py                   │ 加密存于 User.integrations JSON
│  ├─ get_creds(user_id)
│  ├─ set_creds(user_id, [Cred,...])
│  └─ delete_creds(user_id, cred_id)
└────────────────────────────────────────┘
```

### 2.1 加密

- AES-Fernet（`backend/util/encryption.py`）；
- 密钥来自 `ENCRYPTION_KEY` 环境变量；
- 凭证 JSON 在写入 `User.integrations` 前加密、读取后解密。

### 2.2 双锁机制

`creds_manager.acquire()` 关键逻辑：

1. **acquire lock**（短生命周期）：保护一次取凭证 → 用完归还期间，避免 token 在使用中被 refresh 替换；
2. **refresh lock**（更长）：保护多次刷新请求合并 —— 多并发请求只有一个真正去 refresh，其他等结果。

> 没有这套锁的话，并发 OAuth refresh 会拿到不同 token，旧 token 立刻失效，先发出的请求会 401。

### 2.3 Hook

```python
creds_manager.register_creds_changed_hook(callback)
# 凭证刷新 / 用户更新 / 删除 时触发
```

Copilot 在启动时注册 hook，把会话内的 in-flight token 同步替换。

## 3. OAuth

### 3.1 ProviderName

[`backend/integrations/providers.py`](../backend/integrations/providers.py) 是字符串枚举，但**允许动态扩展**：

```python
class ProviderName(str, Enum):
    GITHUB = "github"
    GOOGLE = "google"
    NOTION = "notion"
    ...
```

SDK 用户（包括 Block 作者）可以通过 ProviderBuilder 注册新 provider 字符串，运行时合并到这个枚举里。Pydantic 已配置允许任意 string 值。

### 3.2 BaseOAuthHandler

[`backend/integrations/oauth/_base.py`](../backend/integrations/oauth/_base.py) 定义模板：

```python
class BaseOAuthHandler:
    PROVIDER_NAME: ClassVar[ProviderName]
    DEFAULT_SCOPES: ClassVar[list[str]] = []

    def get_login_url(self, scopes, state, code_verifier=None) -> str: ...
    async def exchange_code_for_tokens(self, code, code_verifier=None) -> OAuth2Credentials: ...
    async def refresh_tokens(self, credentials: OAuth2Credentials) -> OAuth2Credentials: ...
    async def get_user_info(self, access_token: str) -> dict: ...
    async def revoke_token(self, credentials) -> None: ...
```

具体实现按 provider 在 `backend/integrations/oauth/google.py / github.py / ...`。常见模式：标准 PKCE + 标准 token endpoint。

### 3.3 用户视角的 OAuth 流程

```
1. POST /api/integrations/oauth/initiate
       provider: "github"
       scopes: ["repo"]
   →   { auth_url, state }

2. 浏览器跳转 auth_url，用户授权 → callback

3. GET /api/integrations/oauth/callback?code=...&state=...
   ←  服务端 exchange_code_for_tokens()
   ←  写入 User.integrations
   ←  redirect 回前端
```

也支持 `POST /external-api/v1/integrations/oauth/initiate / complete`，用于 SDK 在浏览器外完成（PKCE 的 verifier 由调用方传）。

### 3.4 Token 自动刷新

`creds_manager.acquire()` 取出来的凭证如果 `expires_at` 临近，会自动调对应 handler 的 `refresh_tokens()`。新 token 写回 `User.integrations` 后，并广播 hook 让会话同步。

## 4. Webhook

### 4.1 BaseWebhooksManager

[`backend/integrations/webhooks/_base.py`](../backend/integrations/webhooks/_base.py)

```python
class BaseWebhooksManager:
    PROVIDER_NAME: ClassVar[ProviderName]

    async def get_suitable_auto_webhook(...) -> Webhook:
        """节点级 webhook：复用现有的或创建新的（按 user × resource × events 去重）。"""

    async def get_manual_webhook(...) -> Webhook:
        """preset / 手动绑定的 webhook。"""

    async def _register_webhook(self, credentials, webhook_url, events) -> str:
        """实际调 provider API，返回 provider 侧的 webhook id。"""

    async def _deregister_webhook(self, provider_webhook_id, credentials) -> None: ...

    async def validate_payload(self, payload, secret) -> dict | None:
        """验签并返回标准化 event。"""
```

### 4.2 Ingress 路径

平台暴露统一入口 `POST /webhooks/{webhook_id}`：

1. 取 `IntegrationWebhook` 元数据；
2. 调对应 manager `validate_payload(payload, secret)`；
3. 写入 RabbitMQ webhook 事件；
4. 触发：
   - 若有 AgentNode 绑定（webhook 触发型 Block）：写一条 NodeExecution 入队；
   - 若有 AgentPreset 绑定：按 preset 启动一次 graph 执行。

### 4.3 自动 vs 手动 Webhook

- **自动 webhook**：Block 声明 `webhook_config=BlockWebhookConfig(...)` 时，Builder 在保存图时调 `get_suitable_auto_webhook()` 自动注册（用户无感知）。同 user × resource × events 复用同一 webhook，不重复注册。
- **手动 webhook**：用户在 Library 里给 preset 显式绑定 webhook（"当 GitHub 这个仓库 push 时，跑这个 preset"）。

### 4.4 删除时机

- Block 改了输入参数（resource / events）→ 旧 webhook 反注册，新建一个；
- 用户删除 graph / preset → 检查没人用就反注册；
- 凭证撤销 → 全部 webhook 反注册。

## 5. Managed Providers

部分服务平台想替用户托管，让他们直接用而不需要自己注册 OAuth：

- 典型：Ayrshare（社交媒体聚合发帖）；
- 实现：`backend/integrations/managed_providers/ayrshare.py`；
- 工作机制：平台自己持有一组 root 凭证，按 user 申请 sub-account；后续调用对用户透明。

## 6. Ayrshare 单独说明

[`backend/integrations/ayrshare.py`](../backend/integrations/ayrshare.py)：

- 提供 HTTP 客户端封装；
- 支持视频 / 图文 / 多平台一发；
- 路由暴露在 `backend/api/features/integrations/router.py`，端点形如：
  - `POST /integrations/ayrshare/post`
  - `GET /integrations/ayrshare/accounts`

## 7. 安全

| 风险                   | 缓解                                                                          |
| ---------------------- | ----------------------------------------------------------------------------- |
| 凭证泄漏               | AES 加密 + Postgres 行级访问受 RPC 限制                                       |
| Token 跨用户复用       | `User.integrations` 与 user_id 强绑定；接口都按 `Security(get_user_id)` 校验 |
| Webhook 伪造           | 每个 IntegrationWebhook 持 `secret`，provider manager 必须实现 `validate_payload` |
| OAuth state 篡改       | state 与 code_verifier 服务端短期保存，回调严格校验                           |
| Redirect URI 钓鱼      | 只允许 `FRONTEND_BASE_URL` 下的回调路径                                       |
| Refresh 风暴           | refresh_lock 做合并；失败按指数退避                                            |

## 8. 数据模型对照

| 表 / 字段                       | 内容                                                |
| ------------------------------- | --------------------------------------------------- |
| `User.integrations` (字段)      | 加密 JSON：`[{ id, provider, type, data, ... }]`    |
| `IntegrationWebhook` (表)       | webhook 元数据 + provider 侧 id + secret            |
| `OAuthApplication / *Code / *Token / *RefreshToken` | 平台**作为** OAuth Provider 的对外凭证 |

## 9. 调试技巧

```bash
# CLI 调 OAuth handler（调试用）
poetry run oauth-tool google authorize --redirect-uri http://localhost:3000/...
poetry run oauth-tool google refresh --credentials-id <uuid>

# 看 webhook 是否被注册
SELECT id, provider, "providerWebhookId", events FROM platform."IntegrationWebhook"
WHERE "userId" = '...';

# 验证 ClamAV 与凭证加密
poetry run pytest backend/integrations/creds_manager_test.py
```

## 10. 添加新 provider 实操

详见 [14 Block SDK](./14-block-sdk.md)。简言之：

1. `providers.py` 加枚举值；
2. 写 `OAuthHandler`（如有）；
3. 写 `WebhookManager`（如有）；
4. 用 `ProviderBuilder` 在新 Block 的 `_config.py` 中注册；
5. 写 Block；
6. 加测试。
