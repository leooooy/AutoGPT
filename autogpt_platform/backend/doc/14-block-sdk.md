# 14 Block SDK / ProviderBuilder

## 1. 为什么要有 SDK

Block 体系的复杂点不在"调一次 API"，而在**周边设施**：

- 凭证管理（OAuth Handler、Token 刷新、加密存储）；
- Webhook 注册 / 反注册；
- 成本字典（Block × cost type × 价格）；
- Provider 元数据（图标、显示名、文档链接）。

`backend/sdk/` 把这些抽成一套 **`ProviderBuilder`** 工具链 —— 写新 provider 时不需要散落改十几个文件，"一处定义、全局生效"。

## 2. 目录

```
backend/sdk/
├── __init__.py
├── builder.py         # ProviderBuilder fluent API
├── provider.py        # Provider 数据模型
├── registry.py        # 全局 provider 注册表
└── cost_integration.py  # 成本表合并
```

辅助：
- [`backend/blocks/_static_provider_configs.py`](../backend/blocks/_static_provider_configs.py) —— 内置 provider 的静态声明（图标、产品名等）。
- [`backend/integrations/providers.py`](../backend/integrations/providers.py) —— `ProviderName` 字符串枚举（动态扩展）。

## 3. ProviderBuilder

`ProviderBuilder` 用 fluent 链式 API 描述一个 provider 的所有元数据 + 凭证 + webhook + 成本：

```python
from backend.sdk.builder import ProviderBuilder
from backend.integrations.providers import ProviderName

provider = (
    ProviderBuilder(ProviderName.MY_SERVICE)
    .with_display_name("My Service")
    .with_icon_url("https://...")
    .with_doc_url("https://...")
    .with_oauth(
        handler_class=MyServiceOAuthHandler,         # 见下文
        scopes=["read:user", "write:repo"],
    )
    .with_api_key()                                   # 也允许 API Key
    .with_webhook(
        webhook_type="repo",
        manager_class=MyServiceWebhookManager,
    )
    .with_costs([
        BlockCost(cost_amount=2, cost_type=BlockCostType.RUN, block_class=MyBlock),
    ])
    .build()
)
```

`build()` 会：

1. 在 `registry` 中注册 provider 元数据（前端 / Builder 用）；
2. 把 OAuth handler 加入 `HANDLERS_BY_NAME`；
3. 把 Webhook manager 加入 `WEBHOOK_MANAGERS_BY_PROVIDER`；
4. 把成本表合并进 `BLOCK_COSTS`（`backend/data/block_cost_config.py`）。

### 3.1 OAuth Handler

继承 `BaseOAuthHandler`（[`backend/integrations/oauth/base.py`](../backend/integrations/oauth/base.py)）：

```python
class MyServiceOAuthHandler(BaseOAuthHandler):
    PROVIDER_NAME = ProviderName.MY_SERVICE
    DEFAULT_SCOPES = ["read:user"]

    def get_login_url(self, scopes, state) -> str: ...
    async def exchange_code_for_tokens(self, code, code_verifier) -> OAuth2Credentials: ...
    async def refresh_tokens(self, credentials) -> OAuth2Credentials: ...
    async def get_user_info(self, access_token) -> dict: ...
```

平台已为 GitHub / Google / Notion / Twitter / Discord / HubSpot 等提供 handler，写新的可参考。

### 3.2 Webhook Manager

继承 `BaseWebhooksManager`（[`backend/integrations/webhooks/_base.py`](../backend/integrations/webhooks/_base.py)）：

```python
class MyServiceWebhookManager(BaseWebhooksManager):
    PROVIDER_NAME = ProviderName.MY_SERVICE

    async def _register_webhook(self, credentials, webhook_url, events) -> str:
        """调 provider API 注册 webhook，返回 provider 侧 webhook id。"""
        ...

    async def _deregister_webhook(self, provider_webhook_id, credentials):
        ...

    async def validate_payload(self, payload, secret) -> dict | None:
        """验证签名，返回标准化后的事件。"""
        ...
```

平台暴露 `/webhooks/{webhook_id}`（一个统一入口），路由到对应 provider 的 `validate_payload()`。

### 3.3 静态 Provider 配置

完全没有 OAuth / Webhook 的简单 provider（如某些只用 API Key 的服务）可以走 `_static_provider_configs.py` 直接列表声明，免写 ProviderBuilder。

## 4. 成本表

每个 Block 的成本规则（`backend/data/block_cost_config.py`）：

```python
BLOCK_COSTS: dict[Type[Block], list[BlockCost]] = {
    SendWebRequestBlock: [BlockCost(cost_amount=1, cost_type=BlockCostType.RUN)],
    AnthropicLLMBlock:   [BlockCost(cost_amount=...,  cost_type=BlockCostType.TOKENS)],
    ...
}
```

ProviderBuilder `.with_costs(...)` 会把新条目合并进去，无需手动改这张大表。

## 5. 新增一个 Block 的完整步骤

### 5.1 仅做"调一次 API"的简单 Block

```python
# backend/blocks/my_block.py
import uuid
from backend.blocks._base import Block, BlockCategory, BlockOutput, BlockSchemaInput, BlockSchemaOutput
from backend.data.model import SchemaField

class MyBlock(Block):
    class Input(BlockSchemaInput):
        text: str = SchemaField(description="...")

    class Output(BlockSchemaOutput):
        result: str = SchemaField(description="...")

    def __init__(self):
        super().__init__(
            id="<uuid>",
            description="...",
            categories={BlockCategory.TEXT},
            input_schema=MyBlock.Input,
            output_schema=MyBlock.Output,
        )

    async def run(self, input_data, **kwargs) -> BlockOutput:
        yield "result", input_data.text.upper()
```

放到 `backend/blocks/` 任意位置 → 重启 → 自动可见。

### 5.2 带凭证的 Block（推荐用 SDK）

1. 在 `backend/integrations/providers.py` 加 `MY_SERVICE = "my_service"`；
2. 在 `backend/blocks/my_service/_config.py` 用 `ProviderBuilder` 声明 OAuth handler / API Key 支持；
3. 写 OAuth handler（如有）；
4. Block schema 里加 `credentials: CredentialsMetaInput[ProviderName.MY_SERVICE, Literal["oauth2"]]`；
5. `run()` 里通过 `kwargs["credentials"]` 拿 token 调外部 API；
6. Cost 通过 `.with_costs(...)` 注册。

### 5.3 Webhook 触发型 Block

继续上面的步骤再加：

7. 写 WebhookManager（注册 / 反注册 / 验证）；
8. Block 声明 `webhook_config=BlockWebhookConfig(provider=..., webhook_type="...")`；
9. `run()` 接收 `payload: dict` 输入（webhook 进来时由路由注入）。

## 6. 文件组织约定

每个第三方集成放在 `backend/blocks/<service>/`：

```
backend/blocks/my_service/
├── __init__.py        # 暴露所有 Block 类
├── _config.py         # ProviderBuilder 声明
├── _api.py            # 第三方 API 客户端封装（可选）
├── send_message.py    # 一个 Block
├── list_channels.py   # 另一个 Block
└── ..._test.py
```

OAuth handler / webhook manager 仍放在 `backend/integrations/oauth/` 与 `backend/integrations/webhooks/`，但 SDK 把它们注册到全局表。

## 7. 测试

```bash
# 通用 Block 健康度测试（注册、schema 合法性、UUID 唯一）
poetry run pytest backend/blocks/test/test_block.py -xvs

# 你的 Block 单元测试
poetry run pytest backend/blocks/my_service/send_message_test.py -xvs

# 实际 OAuth handler 试运行
poetry run oauth-tool ...
```

## 8. 写新 Block 时的常见坑

| 坑                                                            | 解决                                                                       |
| ------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `id` 重复或忘记给                                              | 启动时 `load_all_blocks()` 直接抛异常，换一个 `uuid.uuid4()`               |
| 凭证字段名不叫 `credentials` / `*_credentials`                 | 注入逻辑找不到，会 `KeyError`                                              |
| `run()` 写成普通 `async def` 而不是 generator                  | 会被框架检测拒绝；必须 `async def + yield`                                 |
| `output_schema` 没继承 `BlockSchemaOutput`                     | 失去内置 `error` pin                                                       |
| 在 `__init__` 里执行 IO（连 DB / 调 API）                       | 启动会变慢甚至失败；只准声明元数据                                         |
| Block 之间通过 module-level 全局变量共享状态                    | ExecutionManager 多线程下会出错；需要持久化用 DB / Redis                   |
| 忘记给 `categories`                                             | 前端会塞到"未分类"                                                          |
| 用 `print(...)` 输出                                            | 开发可见，CI 看不到；用 `logger`                                           |

## 9. 进一步阅读

- 平台的"Block SDK Guide"：仓库根 `docs/platform/block-sdk-guide.md`；
- 文件上传/媒体处理 Block：根 `docs/platform/workspace-media-architecture.md`；
- ProviderName 已支持的 provider：[`backend/integrations/providers.py`](../backend/integrations/providers.py)。
