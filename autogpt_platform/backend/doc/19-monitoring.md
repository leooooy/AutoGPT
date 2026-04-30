# 19 监控与可观测性

## 1. 三大支柱

| 支柱     | 工具                                       | 模块                                             |
| -------- | ------------------------------------------ | ------------------------------------------------ |
| 指标     | Prometheus                                 | `backend/monitoring/instrumentation.py`          |
| 日志     | autogpt_libs.logging（结构化）+ Cloud 日志 | `backend/util/logging.py`                        |
| 错误追踪 | Sentry                                     | `backend/util/metrics.py` + `sentry-sdk[...]`    |

外加 4 个**周期任务监控器**，由 Scheduler 拉起：

- `accuracy_monitor` —— Block 输出验证；
- `block_error_monitor` —— Block 失败率；
- `late_execution_monitor` —— 执行超时；
- `notification_monitor` —— 通知交付率。

## 2. Prometheus 指标

### 2.1 暴露

- AgentServer 在 `:8006/metrics` 输出（FastAPI instrumentor 自动加）；
- 其它 AppProcess 也可单独 `start_http_server()` 暴露（`backend/util/metrics.py`）。

### 2.2 业务指标

[`backend/monitoring/instrumentation.py`](../backend/monitoring/instrumentation.py)：

| 名称                                              | 类型      | 标签                          | 含义                            |
| ------------------------------------------------- | --------- | ----------------------------- | ------------------------------- |
| `autogpt_graph_executions_total`                  | Counter   | `status`                      | Graph 执行计数（按结束状态）    |
| `autogpt_block_executions_total`                  | Counter   | `block_type`, `status`        | 单 Block 执行计数               |
| `autogpt_block_duration_seconds`                  | Histogram | `block_type`                  | Block 耗时分布                  |
| `autogpt_websocket_connections_total`             | Gauge     | -                             | 实时连接数                      |
| `autogpt_scheduler_jobs`                          | Gauge     | `job_type`, `status`          | 调度任务数                      |
| `autogpt_database_query_duration_seconds`         | Histogram | `operation`, `table`          | DB 查询耗时                     |
| `autogpt_rabbitmq_messages_total`                 | Counter   | `queue`, `status`             | MQ 消息计数                     |
| `autogpt_api_key_usage_total`                     | Counter   | `provider`, `block_type`, `status` | 第三方 API 调用计数        |

`http_*` 是 `prometheus-fastapi-instrumentator` 自动产出的请求级指标（latency / count / size）。

### 2.3 标签策略

- 不放高基数字段（user_id 不出现）；
- block_type 用类名常量；
- queue / table 名是有限集合；
- 异常状态分桶：`success / error / timeout / cancelled`。

## 3. 日志

### 3.1 配置

`backend/util/logging.py`：

- 调 `autogpt_libs.logging.config.configure_logging()`；
- 本地：标准格式 + 颜色；
- 云：JSON Lines，便于聚合（Stackdriver / Datadog / ELK）；
- 每个进程的日志加 `[ServiceName]` 前缀，方便多服务共用 stdout 时区分。

### 3.2 截断

[`backend/util/truncate.py`](../backend/util/truncate.py) 提供 `TruncatedLogger`：

- 默认上限 1000 字符；
- 防止 Block 输入/输出几 MB 内容把日志撑爆；
- 错误现场仍保留前 N 字符 + "...truncated"。

### 3.3 风格

按 `backend/AGENTS.md`：

- `logger.debug("Processing %s items", count)` —— 用 printf 风格，让禁用 debug 时不做字符串拼接；
- `logger.info(f"Processing {count} items")` —— f-string 可读性更强；
- 错误消息 **不要打印完整路径** —— 用 `os.path.basename(...)` 防止泄漏目录结构。

## 4. Sentry

`pyproject.toml`：

```
sentry-sdk = {extras = ["anthropic", "fastapi", "launchdarkly", "openai", "sqlalchemy"], version = "^2.44.0"}
```

集成项：
- FastAPI 自动捕获未处理异常；
- SQLAlchemy 慢查询；
- LaunchDarkly flag 评估；
- OpenAI / Anthropic SDK 抛错。

`backend/util/metrics.py:sentry_init()`：
- 用 `SENTRY_DSN` 初始化；
- `release` / `environment` 来自部署变量；
- AppProcess 异常自动 `sentry_capture_error()`。

## 5. 监控器（Scheduler 周期任务）

### 5.1 accuracy_monitor

[`backend/monitoring/accuracy_monitor.py`](../backend/monitoring/accuracy_monitor.py)

- 抽样：高流量 Store agent 的最近 N 次执行；
- 用 LLM 校验输出是否符合 schema / 描述；
- 累计准确率写 `AnalyticsMetrics`；
- 跌破阈值发 Discord 告警。

### 5.2 block_error_monitor

[`backend/monitoring/block_error_monitor.py`](../backend/monitoring/block_error_monitor.py)

- 扫近 5 分钟 Block 执行失败率；
- 同一 Block × 同一错误连续多次 → 告警；
- 也写 `AnalyticsDetails`。

### 5.3 late_execution_monitor

[`backend/monitoring/late_execution_monitor.py`](../backend/monitoring/late_execution_monitor.py)

- 扫 `AgentGraphExecution` 中 `status=RUNNING` 但 `startedTime > now() - X` 的；
- 标记为 `FAILED(timeout)`；
- 触发 user 通知 + Discord 告警。

### 5.4 notification_monitor

[`backend/monitoring/notification_monitor.py`](../backend/monitoring/notification_monitor.py)

- 扫死信队列堆积 / Postmark 失败率；
- 通知交付率仪表盘数据；
- 异常发 Discord。

## 6. LaunchDarkly Feature Flag

[`backend/util/feature_flag.py`](../backend/util/feature_flag.py)

- 启动时 `initialize()` 拉远端 flag；
- 运行时 `is_enabled(flag, user_context)` 查；
- 关键 flag：
  - `graphiti-memory` —— Copilot 时序记忆；
  - 个别新 Block / 新 API 灰度；
  - 平台计费规则切换。
- 测试用 `NEXT_PUBLIC_PW_TEST=true` 走 mock。

## 7. PostHog

`posthog ^7.6` —— 产品分析事件（前端为主）。后端按需 capture：
- 用户注册 / 订阅升降级 / 大额充值；
- 通过 `User.metadata` 控制是否上报。

## 8. Langfuse / LangSmith

仅用于 Copilot 路径：

- Langfuse 管理系统 prompt + trace LLM 调用（生产推荐）；
- LangSmith 备选；
- 配置：`LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` / `LANGFUSE_HOST`。

## 9. 健康检查

| 端点              | 内容                                     |
| ----------------- | ---------------------------------------- |
| `/health`         | 200 OK 表示进程存活                       |
| `/metrics`        | Prometheus 抓取                           |
| docker healthcheck| `prisma migrate status` / `redis-cli ping` 等 |

## 10. 看板与告警建议

| 关键指标                                          | 推荐告警                                       |
| ------------------------------------------------- | ---------------------------------------------- |
| `autogpt_graph_executions_total{status="error"}`  | 5 分钟变化率 > 阈值                            |
| `autogpt_block_duration_seconds`                  | p95 > 60s                                      |
| `autogpt_websocket_connections_total`             | 突增/突减                                      |
| `autogpt_rabbitmq_messages_total{status="nack"}`  | 持续 > 0                                       |
| `http_request_duration_seconds`                   | p95 > 1s for `/api/graphs/*/execute/*`         |
| Sentry issue 速率                                 | 突增                                            |
| `PlatformCostLog` 成本突增                        | 单 user 单小时 > 阈值（业务报警 SQL）          |
| `PendingHumanReview` 滞留                         | 待审 > 24h                                      |

## 11. 调试技巧

```bash
# 看 Prometheus
curl -s http://localhost:8006/metrics | head -50

# 拉 RabbitMQ queue 深度
docker compose exec rabbitmq rabbitmqctl list_queues name messages

# 查 Sentry trace（在 Sentry 后台按 trace_id 关联多服务）

# 查"卡住"的执行
SELECT id, "executionStatus", "startedTime", now()-"startedTime" AS elapsed
FROM platform."AgentGraphExecution"
WHERE "executionStatus" = 'RUNNING'
ORDER BY "startedTime" LIMIT 20;
```
