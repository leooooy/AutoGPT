# 00 文档说明

## 这是什么

这是一份针对 [`autogpt_platform/backend/`](../) 的中文深度技术文档，目的是让一名新加入的后端工程师在 1~2 天内对整个后端系统有完整、可操作的理解，并知道遇到具体问题时该读哪段代码。

## 后端在做什么（一句话）

**AutoGPT Platform Backend 是一个低代码 / Agent Workflow 平台的服务端**：用户在前端用图形化方式拼装由 Block（功能单元）组成的 Agent，后端负责图的存储、调度、执行、计费、通知，并管理用户、积分、市场（Marketplace）、第三方集成与 Copilot 助手。

后端不是单体进程，而是 9 个进程组成的微服务集群（详见 [03 进程模型](./03-process-model.md)）。

## 阅读顺序建议

```
README → 01 架构总览 ─┬─→ 03 进程模型 → 04 基础设施
                     ├─→ 05 数据模型 → 06 数据访问层
                     ├─→ 07/08/09/10 API 系列
                     ├─→ 11/12/13/14 执行引擎与 Blocks
                     ├─→ 15~18 业务子系统
                     └─→ 19~22 运维与开发
```

不熟悉项目的话先读 README → 01，建立全局心智模型再深入；要做具体改动直接跳到相关章节，每篇都尽量自包含、含必要的交叉链接。

## 命名约定

| 名词              | 含义                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------- |
| Block             | 工作流中的单个功能单元（HTTP 调用、LLM、循环、第三方集成等）。源码位于 `backend/blocks/`。 |
| Graph / AgentGraph| 由 Block 拼接而成的工作流定义，存于 `AgentGraph` + `AgentNode` + `AgentNodeLink` 三张表。 |
| Node              | Graph 中的某个 Block 实例。                                                          |
| Link              | 两个 Node 的输入输出连接（数据流）。                                                  |
| Execution         | 一次工作流运行（`AgentGraphExecution`），包含每个 node 的执行记录（`AgentNodeExecution`）。|
| Library           | 用户私人 Agent 库（`LibraryAgent` 表，含个人收藏与导入）。                           |
| Store             | Agent 市场，发布/审核/下载（`StoreListing` + `StoreListingVersion`）。               |
| Preset            | 预填好输入参数的 Agent 模板，可绑定 Webhook 触发。                                   |
| Credential        | 用户持有的第三方凭证（OAuth Token / API Key），加密存储于 `User.integrations` 字段。 |
| Provider          | 第三方服务（GitHub / Google / Notion / OpenAI 等），定义 OAuth Handler 与 Webhook。  |
| Credit            | 平台内积分单位（实际以微美元 microdollars 存储），用于扣费。                         |
| AppProcess        | 进程基类（`backend/util/process.py`），所有微服务都继承自它。                       |
| AppService        | `AppProcess` 的子类，对外暴露 RPC 接口（基于 Pyro5）。                              |

## 代码引用规则

- 所有相对路径以 `autogpt_platform/backend/` 为根。
- 文件 + 行号写作 `backend/app.py:14`，跨越多行写作 `backend/app.py:14-32`。
- 引用类时写完整路径 `backend.executor.manager.ExecutionManager`。
- 涉及的 Prisma 表名直接用模型名 `AgentGraph`，字段用 `AgentGraph.isActive`。

## 快速校验

如果以下三个问题你都能回答，说明你已经掌握了文档的核心内容：

1. **请求 → 执行 → 通知** 这条主链路涉及哪几个进程？数据怎么流动？（提示：见 [01](./01-architecture-overview.md) 末尾时序图）
2. 一个 Block 从代码到出现在前端 UI，需要做什么？被执行时调用栈是怎样的？（提示：[13](./13-blocks-system.md) + [11](./11-execution-engine.md)）
3. 用户余额扣费是同步还是异步？什么时候 commit？怎么避免负余额竞态？（提示：[16](./16-credit-billing.md)）
