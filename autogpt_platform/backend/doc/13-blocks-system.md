# 13 Blocks 体系

## 1. 是什么

**Block 是工作流的最小可执行单元**，对应一段功能：HTTP 请求、LLM 调用、文本拼接、Notion 写入、循环、条件分支……

- 用户在 Builder 里把 Block 拖到画布上，拼成 Graph；
- 后端在 ExecutionManager 里逐节点（每个 node 持有一个 Block class 实例）调用 `block.execute()` 跑出输出；
- 输出经 link 传给下游 Block 的输入。

源码根目录：[`backend/blocks/`](../backend/blocks/)，含 100+ 个 Python 文件 / 50+ 个第三方集成子目录。

## 2. 基类与 Schema

### 2.1 `Block` 基类

[`backend/blocks/_base.py`](../backend/blocks/_base.py)

```python
class Block(Generic[BlockSchemaInputT, BlockSchemaOutputT], ABC):
    id: str                          # UUID（代码硬编码、不可变）
    description: str
    input_schema: Type[BlockSchemaInput]
    output_schema: Type[BlockSchemaOutput]
    categories: set[BlockCategory]
    costs: list[BlockCost] = []
    disabled: bool = False
    static_output: bool = False
    webhook_config: BlockWebhookConfig | None = None
    is_sensitive_action: bool = False

    async def run(self, input_data: Input, **kwargs) -> BlockOutput:
        """子类实现：async generator yield (output_pin, value)。"""
        raise NotImplementedError

    async def execute(self, input_data, **kwargs) -> BlockOutput:
        """统一包装：审核、输入校验、调用 run()、输出校验。"""
        ...
```

### 2.2 `BlockSchema`

```python
class BlockSchemaInput(BaseModel): ...
class BlockSchemaOutput(BaseModel):
    error: str = ""        # 内置：异常时填入错误描述
```

输入字段必须用 `SchemaField(...)` 装饰器声明（来自 `backend.data.model`），用于：

- 默认值、描述、UI 渲染提示；
- 是否敏感（`secret=True`）；
- 凭证字段类型 `CredentialsMetaInput[ProviderName, FieldType]`；
- 多选 / 枚举提示。

### 2.3 BlockCategory

- `AI`、`LOGIC`、`COMMUNICATION`、`SOCIAL`、`PRODUCTIVITY`、`DATA`、`DEVELOPER_TOOLS`、`INPUT`、`OUTPUT`、`AGENT`、`MULTIMEDIA`、`SEARCH`、`HARDWARE`、`SAFETY`、`HUMAN_INPUT`、`TEXT`…

UI 按 category 分组。前端搜索框的"分类筛选"对应这个枚举。

## 3. 自动注册

### 3.1 启动扫描

[`backend/blocks/__init__.py`](../backend/blocks/__init__.py) 暴露 `load_all_blocks()`：

1. 递归扫描 `backend/blocks/` 下所有 `.py`；
2. `importlib.import_module()` 每个模块；
3. 收集所有 `Block` 子类；
4. 校验：
   - 类名以 `Block` 结尾；
   - 抽象基类以 `Base` 结尾才被忽略；
   - `id` 必须是 36 字符 UUID；
   - 全局 id 唯一；
5. 缓存 1 小时（避免反复扫描）。

### 3.2 同步到数据库

[`backend/data/block.py:initialize_blocks()`](../backend/data/block.py)：

- 遍历所有 Block，把 `(id, name, inputSchema, outputSchema, description)` upsert 到 `AgentBlock` 表；
- 删除已不存在的；
- 在每个 `AppProcess.lifespan` 启动时调用一次（DatabaseManager / AgentServer / ExecutionManager 都跑）。

> 这意味着**新增一个 Block 文件不需要写 SQL 也不需要单独注册**：把它放到 `backend/blocks/` 任意位置，重启后自动被发现。

## 4. 典型分类速览

| 分类           | 代表 Block / 文件                                                   | 说明                                            |
| -------------- | ------------------------------------------------------------------- | ----------------------------------------------- |
| 控制流         | `branching.py` (`ConditionBlock`, `IfInputMatchesBlock`)            | if / switch                                     |
| 迭代           | `iteration.py` (`StepThroughItemsBlock`)                            | for-each 列表 / 字典                            |
| 基础           | `basic.py` (`StoreValueBlock`, `PrintToConsoleBlock`, `NoteBlock`) | 缓存值、调试、注释                              |
| LLM            | `llm.py` (多 provider)                                              | OpenAI / Anthropic / Groq / Ollama / OpenRouter |
| HTTP           | `http.py` (`SendWebRequestBlock`)                                   | GET/POST/PUT/DELETE                             |
| 嵌套 agent     | `agent.py` (`AgentExecutorBlock`)                                   | 调另一个 graph                                   |
| 文本           | `text.py`、`count_words_and_char_block.py`、`code_extraction_block.py` | 拼接、提取、统计                              |
| 数据处理       | `data_manipulation.py`、`spreadsheet.py`、`xml_parser.py`、`encoder/decoder_block.py` | JSON / CSV / XML / Base64                |
| 多媒体         | `ai_image_generator_block.py`、`ai_music_generator.py`、`text_to_speech_block.py`、`talking_head.py`、`video/`、`flux_kontext.py`、`ideogram.py` | 图像、音频、视频生成与处理 |
| 搜索           | `search.py`、`exa/`、`firecrawl/`、`perplexity.py`、`google/`、`google_maps.py` | 各种搜索 API                          |
| 第三方集成     | `github/`、`google/`、`notion/`、`linear/`、`hubspot/`、`discord/`、`slack`（在 `slant3d` 等子目录）、`twitter/`、`telegram/`、`reddit.py`、`apollo/`、`airtable/`、`hubspot/`、`mem0.py`、`pinecone.py`、`compass/`、`zerobounce/`、`smartlead/`、`stagehand/`、`fal/`、`replicate/` | 50+ 服务，每个子目录独立 |
| 自动化         | `autopilot.py`                                                      | LLM 自主规划执行                                |
| HITL           | `human_in_the_loop.py` (`HumanReviewBlock`)                         | 暂停等待人工                                    |
| MCP            | `mcp/block.py`                                                      | Model Context Protocol 工具调用                 |
| 编码沙箱       | `code_executor.py`                                                  | E2B / 代码解释器                                |
| 系统           | `system/`、`time_blocks.py`、`maths.py`、`sampling.py`              | 时间、数学、采样工具                            |
| 持久化         | `persistence.py`、`sql_query_block.py`                              | 跨节点存值、SQL 查询                            |
| Webhook        | `generic_webhook/`                                                  | 通用 webhook 触发器                             |

## 5. Block 的执行签名

```python
class MyBlock(Block):
    class Input(BlockSchemaInput):
        text: str = SchemaField(description="原文")
        upper: bool = SchemaField(default=False)
        credentials: CredentialsMetaInput[
            ProviderName.OPENAI, Literal["api_key"]
        ] = SchemaField(description="OpenAI Key")

    class Output(BlockSchemaOutput):
        result: str = SchemaField(description="处理结果")

    def __init__(self):
        super().__init__(
            id="b1234567-89ab-cdef-0123-456789abcdef",
            description="Demo block",
            categories={BlockCategory.TEXT},
            input_schema=MyBlock.Input,
            output_schema=MyBlock.Output,
            costs=[BlockCost(cost_amount=1, cost_type=BlockCostType.RUN)],
        )

    async def run(self, input_data: Input, **kwargs) -> BlockOutput:
        creds = kwargs["credentials"]                         # 已注入
        ctx = kwargs["execution_context"]                      # ExecutionContext
        out = input_data.text.upper() if input_data.upper else input_data.text
        yield "result", out
```

`kwargs` 中的常见键：

- 凭证字段同名键（`credentials`、`openai_credentials`...）—— 已经过 acquire/refresh；
- `execution_context: ExecutionContext` —— `user_id` / `graph_id` / `node_id` / `graph_exec_id` / `node_exec_id` / 当前 workspace；
- `execution_processor` —— 给极少数需要写额外日志/计费的 Block 用；
- 其他 graph 注入：例如 `current_workspace_session_id`。

## 6. 输出与多值流

`run()` 是一个 async generator，可以：

```python
yield "result", value          # 单一输出
yield "list_item", item        # 在循环里多次 yield → 多值流
yield "error", "..."           # 标准错误 pin（也能 raise）
```

每次 `yield` 都会写一条 `AgentNodeExecutionInputOutput` 记录。下游节点的 link：

- 静态（`isStatic=true`）：每个值都触发一次下游执行；
- 动态：聚合后再触发。

## 7. 凭证字段约定

输入 schema 里命名为 `credentials` 或 `*_credentials`，类型必须是 `CredentialsMetaInput[ProviderName, FieldType]`。

`ProviderName` 是动态扩展的字符串枚举（[`backend/integrations/providers.py`](../backend/integrations/providers.py)），SDK 用户也可以注册新 provider（如 MCP）。

支持的字段类型：

- `Literal["api_key"]` —— 静态 API Key；
- `Literal["oauth2"]` —— OAuth Token（自动刷新）；
- `Literal["webhook"]` —— webhook secret；
- `Literal["host_scoped"]` —— 主机粒度凭证。

## 8. Webhook 触发型 Block

Block 可以声明自己"被 webhook 触发"：

```python
super().__init__(
    ...,
    webhook_config=BlockWebhookConfig(
        provider=ProviderName.GITHUB,
        webhook_type="repo",
        ...
    ),
)
```

代表：当用户使用这个 Block 时，平台会自动为其注册一个 webhook 到 GitHub；事件到达时该 Block 输出对应数据。详见 [15 集成与 OAuth](./15-integrations-oauth.md)。

## 9. 敏感操作 / Auto Mod

`is_sensitive_action=True` 的 Block（发邮件、创建文件、删除资源等）会进入 Auto Mod 流程：

- 代码侧：在 `Block.execute()` → `_execute()` 里走规则引擎；
- 不通过 → 自动 yield 一个 `PendingHumanReview`，状态 `REVIEW`；
- 用户审批后才会真正执行。

## 10. 测试

- 通用测试：`backend/blocks/test/test_block.py` —— 自动遍历所有 Block，检查注册、I/O schema、UUID 唯一；
- 跑特定 Block：
  ```bash
  poetry run pytest 'backend/blocks/test/test_block.py::test_available_blocks[GetCurrentTimeBlock]' -xvs
  ```
- 单 Block 自带 `*_test.py`：例如 `ai_condition_test.py`、`sql_query_block_test.py`，用于真实 input/output 验证。

## 11. UI 暴露

前端通过 `GET /api/blocks` 拿到全部 Block 元数据（schema、category、cost、icon）。Storybook 与 Builder 都基于这份数据动态渲染 —— **不需要前端硬编码 Block 类型**。

## 12. 何时新增 vs 复用

新增一个 Block 之前，问自己：

1. 是否能用现有的 `SendWebRequestBlock` + 几个数据处理 Block 拼出来？能就拼。
2. 是否会成为 LLM-graph builder 中"可被推荐"的高频功能？是 → 独立 Block 更友好。
3. 是否需要凭证或 webhook？是 → 走 SDK ProviderBuilder（[14 Block SDK](./14-block-sdk.md)）。

新增 Block 的实操步骤见 [14 Block SDK](./14-block-sdk.md)。
