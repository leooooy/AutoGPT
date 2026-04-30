# 05 数据模型（Prisma Schema）

> Schema 文件：[`backend/schema.prisma`](../schema.prisma)。本文按业务域组织，重点说明字段含义与表间关系，**不重复 schema 中的字段类型声明**，需要细节请直接看 schema。

## 1. 业务域分组速览

| 域                        | 主要表                                                                                                          |
| ------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 用户与认证                | `User`, `UserBalance`, `UserOnboarding`, `UserWorkspace`, `APIKey`, `OAuthApplication/AuthorizationCode/AccessToken/RefreshToken` |
| Agent 图与执行            | `AgentBlock`, `AgentGraph`, `AgentNode`, `AgentNodeLink`, `AgentGraphExecution`, `AgentNodeExecution`, `AgentNodeExecutionInputOutput`, `AgentPreset`, `AgentExecutionScheduledJob`, `PendingHumanReview` |
| Library                   | `LibraryAgent`, `LibraryFolder`                                                                                 |
| Store / Marketplace       | `StoreListing`, `StoreListingVersion`, `StoreListingReview`, `Profile`                                          |
| 集成与 Webhook            | `IntegrationWebhook`                                                                                            |
| 通知与 Push               | `UserNotificationBatch`, `NotificationEvent`, `PushSubscription`                                                |
| 计费                      | `CreditTransaction`, `CreditRefundRequest`, `PlatformCostLog`                                                   |
| Copilot                   | `ChatSession`, `CoPilotUnderstanding`                                                                           |
| Builder / Search          | `BuilderSearchHistory`, `UnifiedContentEmbedding`                                                               |
| Platform Linking          | `PlatformLink`, `PlatformUserLink`                                                                              |
| Analytics                 | `AnalyticsDetails`, `AnalyticsMetrics`                                                                          |

## 2. 用户与认证

### `User`（核心账户）

- `id` 与 Supabase user id 一致；
- `integrations` 字段：**所有第三方凭证以加密 JSON 字符串存储于此**（不是单独表）。读写经 `backend/integrations/credentials_store.py` 加解密；
- `subscriptionTier`：`NO_TIER / BASIC / PRO / MAX / BUSINESS / ENTERPRISE`，决定 Copilot 限流倍率（1×/5×/20×/60×/60×）；当前默认 `PRO`，GA 时改回 `BASIC` + 后台 backfill；
- `metadata`：JSON，存放迁移/标志位等不规范字段；
- `notify*` 字段：通知偏好开关；
- `topUpConfig`：自动充值阈值/金额配置。

关系：1 User → N AgentGraph / Execution / LibraryAgent / Profile / ChatSession ……

### `UserBalance`

- 1:1 与 `User`；
- `balance` 用 **微美元**（int），1 USD = 1,000,000 microdollars；
- 是 `CreditTransaction` 流水的快照（最终一致），扣费同事务更新；
- 详见 [16 积分与计费](./16-credit-billing.md)。

### `APIKey`

- 给 SDK / 第三方调用用；
- `permissions[]` 控制粒度（IDENTITY / READ_BLOCK / EXECUTE_GRAPH / WRITE_GRAPH / MANAGE_INTEGRATIONS …）；
- 在 External v1 API 经 `require_permission(*APIKeyPermission)` 校验。

### `OAuthApplication / OAuthAuthorizationCode / OAuthAccessToken / OAuthRefreshToken`

- 平台**作为 OAuth Provider**（让第三方扮演用户调 AutoGPT API）；
- 标准 OAuth 2.0 + PKCE：
  - AuthorizationCode TTL 10 分钟；
  - AccessToken TTL 1 小时；
  - RefreshToken TTL 30 天。

### `UserOnboarding` / `UserWorkspace`

- 引导流程：以 `OnboardingStep` 枚举推进（28 步）；记录用户偏好如 `usageReason`、`integrations`；
- 工作区：用户私有文件存储，挂载 `UserWorkspaceFile` 子表（path / storagePath / mimeType / sizeBytes / checksum / isDeleted）。

## 3. Agent 图与执行

### Block 元数据

`AgentBlock` 表是从代码扫描出来的 Block 元数据缓存：
- `id` = Block 类的 UUID（代码中硬编码）；
- `name` 唯一；
- `inputSchema` / `outputSchema`：来自 `Block.input_schema/output_schema` 的 JSON Schema；
- `optimizedDescription`：LLM 生成的简化描述，供 Copilot/Builder 搜索使用。

启动时由 `backend/data/block.py:initialize_blocks()` 同步：删除已不存在的、插入新增的、更新有变化的。

### `AgentGraph`（工作流定义）

- **复合主键 `(id, version)`**：每次重大修改会发布新 version（旧 version 仍在数据库里供历史执行回放）；
- `isActive`：标记当前默认运行的版本（同一 graph 同一时间只有一个 isActive=true）；
- `forkedFromId/Version`：fork 链；
- 关系：→ `AgentNode[]` → `AgentNodeLink[]` → `AgentPreset[]` → `StoreListingVersion[]`。

### `AgentNode` / `AgentNodeLink`

- `AgentNode.agentBlockId` → `AgentBlock`；`agentGraphId/version` 指回 graph；
- `constantInput` JSON：节点的"硬编码输入"（用户在 Builder 里填的常量）；
- `metadata` JSON：UI 元数据（坐标、注释等）；
- `AgentNodeLink`：source pin `(node.sourceName)` → sink pin `(node.sinkName)`；
- `isStatic`：true 表示该输出可以被多个下游消费多次（典型如配置常量）；false 是单次数据流。

### 执行三表

```
AgentGraphExecution        ──┐
  ├─ inputs            JSON │ 1:N
  ├─ credentialInputs  JSON │
  ├─ nodesInputMasks   JSON │
  ├─ stats             JSON │
  ├─ executionStatus   enum │
  ├─ shareToken        str? │
  └─ parentGraphExecutionId?│  ← 嵌套 graph 调用
                            ▼
        AgentNodeExecution            ──┐
          ├─ executionStatus   enum    │ 1:N
          ├─ executionData     JSON    │
          ├─ addedTime,queuedTime,    │
          │   startedTime,endedTime    │
          └─ stats             JSON    │
                                       ▼
                AgentNodeExecutionInputOutput
                  ├─ name (pin 名)
                  ├─ data JSON
                  └─ time
```

- `executionStatus` 取值：`INCOMPLETE → QUEUED → RUNNING → COMPLETED / FAILED / REVIEW`；
- `nodesInputMasks` 用于在重跑时屏蔽某些输入；
- `stats`：`{node_count, total_cost_microdollars, duration_seconds, llm_token_count, ...}`。
- 复合唯一约束让 `(graphExecId, nodeId, pin name)` 仅有一个 input 记录，但**输出可以多条**（多值流）。
- 关键索引：`(userId, isDeleted, createdAt)` 给"我的执行历史"加速；`(createdAt)` 给监控扫描用。

### `AgentPreset`

- 给一个 Graph 预填一组输入 + 凭证 + 可选的 `webhookId`；
- 当 webhook 触发时，按 preset 启动一次 graph 执行；
- 是 Library 暴露给最终用户的"开箱即用"形态。

### `AgentExecutionScheduledJob`

- 用户配置的定时 Graph 执行；
- 字段：`cron`、`agentGraphId/version`、`inputs`、`isActive`、`nextRunAt`；
- 由 Scheduler 进程在 APScheduler 里 mirror 一份。

### `PendingHumanReview`

- HITL（Human-in-the-Loop）必备：某个 node 暂停，等待人工批准/编辑；
- 主键 `nodeExecId`；
- 关键字段：`status` (WAITING/APPROVED/REJECTED)、`payload`、`instructions`、`editable`、`reviewMessage`、`wasEdited`、`processed`、`reviewedAt`；
- 特殊 key `auto_approve_{graphExecId}_{nodeId}`：表示已被预先 auto-approve；
- `cancel_pending_reviews_for_execution()`：execution 失败时级联取消该图的所有等待项。

## 4. Library

### `LibraryAgent`

- 用户私人库中的"代理实例"，绑定具体 `agentGraphId/version`；
- `isFavorite`、`isCreatedByUser`、`isArchived`、`isDeleted`；
- 复合唯一 `(userId, agentGraphId, agentGraphVersion)`：同一个 graph 版本只入库一次；
- 可关联 `folderId` 进入文件夹。

### `LibraryFolder`

- 树形结构（`parentId` self-reference）；
- 软删除（`isDeleted`）。

## 5. Store / Marketplace

### `StoreListing` / `StoreListingVersion`

- `StoreListing` 是 listing 元数据壳：`slug`、`agentGraphId`（唯一，一个 graph 只能有一个 listing）、`owningUserId`、`activeVersionId`、`useForOnboarding`；
- `StoreListingVersion` 是真实可被审核/购买的版本：包含 `version`、`submissionStatus` (DRAFT / PENDING / APPROVED / REJECTED)、`reviewerId`、`categories[]`、`imageUrls[]`；
- 复合唯一 `(storeListingId, version)`。

### `StoreListingReview`

- 用户对某个 listing version 的评分与评论；
- 复合唯一 `(storeListingVersionId, reviewByUserId)` 防重复评论。

### `Profile`

- 创作者公开主页：username（唯一）、description、links[]、avatarUrl、isFeatured。

### `UnifiedContentEmbedding`

- pgvector 表，存 Store agent / creator 的语义向量；
- 复合索引 `(contentType, userId)` + 向量 ANN 索引；
- 由 `backend/api/features/store/embeddings.py` 维护。

## 6. 集成与 Webhook

### `IntegrationWebhook`

- 字段：`userId`、`provider`、`credentialsId`、`webhookType`、`resource`、`events[]`、`config` JSON、`secret`、`providerWebhookId`；
- 与 `AgentNode` / `AgentPreset` 多对多绑定（被关联即"哪个节点/预设监听该 webhook"）；
- 由 `backend/integrations/webhooks/` 各 provider 实现真正的注册/注销。

## 7. 通知与 Push

### `UserNotificationBatch` + `NotificationEvent`

- 一对多：每个 user × 每种 `NotificationType` 只有一个活跃 batch；
- 内部聚合若干 `NotificationEvent`；
- 用于摘要类通知（每日/每周/批量错误）；立即类直接走 RabbitMQ + 邮件。

### `PushSubscription`

- 每个浏览器订阅一条；
- 字段：`endpoint`（唯一）、`p256dh`、`auth`、`userAgent`、`failCount`、`lastFailedAt`；
- 失败 5 次后自动下线。

## 8. 计费

### `CreditTransaction`

- **复合主键 `(transactionKey, userId)`**：`transactionKey` 是幂等键（业务侧生成的 UUID 等），重复提交不会重复扣费；
- `type` 枚举：`TOP_UP / USAGE / GRANT / REFUND / CARD_CHECK / SUBSCRIPTION`；
- `runningBalance`：扣减后的余额（**冗余写入**，便于审计 & 防漂移）；
- `metadata` JSON：业务上下文（USAGE 会塞 `graph_exec_id / block / input` 等）；
- 索引 `(userId, createdAt)`。

### `CreditRefundRequest`

- 用户发起的退款申请，状态：`PENDING / APPROVED / REJECTED`。

### `PlatformCostLog`

- **平台对外侧成本**（自家烧的钱），不直接等于用户被扣的额度；
- 字段：`provider`（小写标准化：openai / anthropic / replicate ...）、`costMicrodollars`、`inputTokens`、`outputTokens`、`cacheReadTokens`、`cacheCreationTokens`、`trackingType`、`trackingAmount`、关联 `graphExecId / nodeExecId`；
- 多个索引支持成本分析（按 provider、按 user、按时间窗口聚合）。

## 9. Copilot

### `ChatSession`

- 一次"AI 助手会话"；
- 关键字段：`title`、`credentials` JSON（ephemeral 凭证快照）、`metadata`、`successfulAgentRuns` JSON（按 graph_id 计数）、`totalPromptTokens`、`totalCompletionTokens`；
- 关联 `User`。

### `CoPilotUnderstanding`

- 1:1 用户级"业务理解"配置（让 Copilot 知道用户/团队是做什么的）；
- 用于注入到系统提示。

## 10. Platform Linking

### `PlatformLink` / `PlatformUserLink`

- `PlatformLink`：服务器/工作区级别的链接（一个 Discord guild 绑定到 AutoGPT 用户）；
- `PlatformUserLink`：DM / 个人级别的链接（一个 Discord 用户绑定到 AutoGPT 用户）；
- `platform` 枚举：`Discord / Slack / Telegram / ...`；
- token TTL 短（默认 3 小时），用于授权流程。

## 11. 关键索引策略小结

| 表                              | 关键索引                                                  | 加速场景                          |
| ------------------------------- | --------------------------------------------------------- | --------------------------------- |
| `AgentGraphExecution`           | `(userId, isDeleted, createdAt)`                          | "我的执行历史"分页                |
| `AgentGraphExecution`           | `(createdAt)`                                             | 后台扫描超时执行                  |
| `AgentNodeExecution`            | `(agentGraphExecutionId, agentNodeId, executionStatus)`   | 单次执行的节点状态查询            |
| `AgentNodeExecutionInputOutput` | `(nodeExecutionId, name, time)` 复合唯一                  | upsert input；多值 output         |
| `CreditTransaction`             | `(userId, createdAt)`                                     | 余额历史                          |
| `PlatformCostLog`               | `(provider, createdAt)` / `(userId, createdAt)`           | 成本分析                          |
| `UnifiedContentEmbedding`       | `(contentType, userId)` + pgvector                        | 向量召回 + 过滤                   |

## 12. 看 schema 的入口

```bash
# 直接看
$EDITOR backend/schema.prisma

# 重新生成 Python client
poetry run prisma generate

# 应用迁移（dev）
poetry run prisma migrate dev

# 应用迁移（prod / docker）
poetry run prisma migrate deploy
```
