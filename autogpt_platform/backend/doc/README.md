# AutoGPT Platform Backend 文档

> 面向 AutoGPT Platform 后端代码（`autogpt_platform/backend/`）的深度技术文档。
> 适用于希望理解后端架构、二次开发、运维部署的工程师。

## 文档导航

### 总览
- [00 文档说明](./00-overview.md) — 文档结构与阅读顺序
- [01 架构总览](./01-architecture-overview.md) — 分层架构、模块依赖、整体数据流
- [02 技术栈](./02-tech-stack.md) — 框架、库、外部服务清单
- [03 进程模型与服务拓扑](./03-process-model.md) — 9 个 AppProcess 的职责与通信

### 基础设施与数据
- [04 基础设施依赖](./04-infrastructure.md) — PostgreSQL / Redis Cluster / RabbitMQ / ClamAV / Supabase / GCS
- [05 数据模型](./05-data-model.md) — Prisma schema 全量解读
- [06 数据访问层](./06-data-layer.md) — `backend/data/` 各模块职责

### API 层
- [07 REST API 总览](./07-api-rest.md) — `AgentServer` 启动、生命周期、中间件
- [08 API 路由清单](./08-api-routes.md) — 各 feature 主端点速查表
- [09 WebSocket API](./09-api-websocket.md) — 订阅协议、事件类型、连接管理
- [10 External API](./10-api-external.md) — 面向第三方的 v1 API 与 API Key 鉴权

### 执行引擎与 Blocks
- [11 执行引擎](./11-execution-engine.md) — `ExecutionManager` / `ExecutionProcessor` 时序
- [12 调度器](./12-scheduler.md) — APScheduler 持久化定时任务
- [13 Blocks 体系](./13-blocks-system.md) — Block 基类、自动注册、典型分类
- [14 Block SDK](./14-block-sdk.md) — `ProviderBuilder`、Provider 注册、新增 Block 流程

### 业务子系统
- [15 集成与 OAuth](./15-integrations-oauth.md) — Credentials Manager、OAuth Handler、Webhook
- [16 积分与计费](./16-credit-billing.md) — Credit 流水、动态计费、Stripe 订阅
- [17 Copilot](./17-copilot.md) — 双执行链（SDK / Baseline）、Bot Bridge、限流
- [18 通知系统](./18-notifications.md) — RabbitMQ 队列分层、邮件 / Web Push 模板

### 可观测性与开发运维
- [19 监控与可观测性](./19-monitoring.md) — Prometheus 指标、Sentry、Late Execution Monitor
- [20 本地开发与启动](./20-dev-setup.md) — Poetry / Docker / Prisma migrate
- [21 测试策略](./21-testing.md) — pytest / snapshot / colocated 测试
- [22 部署](./22-deployment.md) — Dockerfile、docker-compose、迁移策略

## 快速入口

| 你想…                                | 先读                                                                |
| ------------------------------------ | ------------------------------------------------------------------- |
| 理解后端整体设计                     | [01 架构总览](./01-architecture-overview.md)                        |
| 增加一个 Block                       | [13 Blocks 体系](./13-blocks-system.md) → [14 Block SDK](./14-block-sdk.md) |
| 增加一个 REST 端点                   | [07 REST API 总览](./07-api-rest.md)                                |
| 调试某次执行                         | [11 执行引擎](./11-execution-engine.md) → [05 数据模型](./05-data-model.md) |
| 接入新的第三方 OAuth                 | [15 集成与 OAuth](./15-integrations-oauth.md)                       |
| 在本地把后端跑起来                   | [20 本地开发与启动](./20-dev-setup.md)                              |
| 写测试                               | [21 测试策略](./21-testing.md)                                      |

## 写作约定

- 所有路径相对 `autogpt_platform/backend/` 根目录。
- 行号引用形如 `backend/app.py:14`。
- 文档基于 2026-04-30 时点的 `master` 分支代码（commit `c08b9774d` 附近）。代码持续演进，行号可能漂移，但模块职责相对稳定。
