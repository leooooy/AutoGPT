# 09 WebSocket API

## 1. 进程与端口

WebSocket 由 `WebsocketServer` 独立进程承载（[`backend/api/ws_api.py`](../backend/api/ws_api.py)），默认端口 `8001`，路径以 `/ws/...` 开头。

```python
class WebsocketServer(AppProcess):
    service_name = "WebsocketServer"
    def run(self):
        uvicorn.run(
            "backend.api.ws_api:app",
            host=settings.WEBSOCKET_HOST,
            port=settings.WEBSOCKET_PORT,    # 8001
        )
```

把 WebSocket 与 REST 分开成独立进程是为了：

- WebSocket 是**长连接**，单连接会占住一个 worker；与短连接 REST 隔离能避免互相饿死；
- 容器水平扩展更直观；
- 升级/重启策略可以不同（WS 重启会断所有连接，所以更倾向于灰度）。

## 2. 路径

| 路径                                | 说明                                          |
| ----------------------------------- | --------------------------------------------- |
| `/ws/{graph_exec_id}?token=<JWT>`   | 订阅一次具体执行的事件                        |
| `/ws/notifications?token=<JWT>`     | 订阅用户的通知（任意类型）                    |

> 鉴权 token 通过 query 参数传递（部分浏览器 / SDK 不能在 WS handshake 加 Authorization 头）。

## 3. 鉴权流程

1. handshake 之前，从 query string 提取 `token`；
2. `get_jwt_payload(token)` 解析，失败按以下码关闭：
   - `4001`：缺 token
   - `4002`：解析失败 / 签名不对
   - `4003`：用户不存在 / 权限不够
3. 鉴权通过后，把 `(user_id, websocket)` 注册到 `ConnectionManager`。

> 关闭码 `4xxx` 系列是应用层自定义（per RFC 6455）。

## 4. 消息协议

### 4.1 通用包装

[`backend/api/model.py`](../backend/api/model.py)

```python
class WSMessage(BaseModel):
    method: WSMethod          # 必填
    data: dict | None = None  # 业务负载
    success: bool = True
    channel: str | None = None
    error: str | None = None
```

### 4.2 `WSMethod` 枚举

```python
class WSMethod(str, Enum):
    SUBSCRIBE_GRAPH_EXEC   = "subscribe_graph_execution"   # client→server
    SUBSCRIBE_GRAPH_EXECS  = "subscribe_graph_executions"  # client→server
    UNSUBSCRIBE            = "unsubscribe"                 # client→server
    GRAPH_EXECUTION_EVENT  = "graph_execution_event"       # server→client
    NODE_EXECUTION_EVENT   = "node_execution_event"        # server→client
    NOTIFICATION           = "notification"                # server→client
    ERROR                  = "error"                       # server→client
    HEARTBEAT              = "heartbeat"                   # 双向
```

### 4.3 客户端 → 服务端

```jsonc
// 订阅一次具体执行
{
  "method": "subscribe_graph_execution",
  "data": { "graph_exec_id": "uuid" }
}

// 订阅某个 graph 的全部执行
{
  "method": "subscribe_graph_executions",
  "data": { "graph_id": "uuid" }
}

// 取消订阅
{
  "method": "unsubscribe",
  "data": { "channel": "graph_exec/uuid" }
}
```

### 4.4 服务端 → 客户端

```jsonc
// graph 状态更新
{
  "method": "graph_execution_event",
  "channel": "graph_exec/uuid",
  "data": {
    "graph_exec_id": "...",
    "status": "RUNNING",
    "stats": { "node_count": 10, "completed": 7, "duration_seconds": 12.3 }
  }
}

// 单 node 状态更新
{
  "method": "node_execution_event",
  "channel": "graph_exec/uuid",
  "data": {
    "graph_exec_id": "...",
    "node_exec_id": "...",
    "node_id": "uuid",
    "status": "COMPLETED",
    "output": { "pin": "value", ... }
  }
}

// 通知
{
  "method": "notification",
  "data": { "type": "AGENT_RUN", "..." }
}

// 错误
{
  "method": "error",
  "success": false,
  "error": "Subscription denied: ..."
}

// 心跳（每 30s 一次）
{ "method": "heartbeat" }
```

## 5. ConnectionManager

[`backend/api/conn_manager.py`](../backend/api/conn_manager.py)

```
ConnectionManager
  ├─ active_connections: dict[user_id, set[WebSocket]]
  ├─ subscriptions: dict[channel, set[WebSocket]]
  └─ _Subscription[]:  Redis Sharded PubSub 客户端（按 channel 分片）
```

职责：

1. 接受 `subscribe_*` 消息：把 (user_id, websocket) 加入对应 channel；
2. 启动一个 Redis `SSUBSCRIBE`，把消息搬到对应 websocket；
3. 心跳：定时给所有连接发 `HEARTBEAT`；
4. 断线/异常时清理订阅与 Redis 连接；
5. **MOVED 重定向**：Cluster 拓扑变化时，重新选 shard 节点；
6. 重连：指数退避（0.5s → 8s，超时 60s）。

## 6. 事件源

WebSocket 不是事件的产生者，而是 Redis 上事件的消费者。事件由其他进程通过 [`backend/data/event_bus.py`](../backend/data/event_bus.py) 发布：

```python
# ExecutionManager 写入 Redis
event_bus.publish(
    channel=f"execution_event_bus_name/{graph_id}/{exec_id}",
    message=NodeExecutionEvent(...),
)

# WebsocketServer 自动 pubsub 转发
```

支持事件类型：

| 来源                                 | 事件                                               | 推到客户端                |
| ------------------------------------ | -------------------------------------------------- | ------------------------- |
| ExecutionManager                     | `ExecutionEventType.GRAPH_EXEC_UPDATE`             | `GRAPH_EXECUTION_EVENT`   |
| ExecutionManager                     | `ExecutionEventType.NODE_EXEC_UPDATE`              | `NODE_EXECUTION_EVENT`    |
| NotificationManager / data.notification_bus | `NotificationEvent`                          | `NOTIFICATION`            |

## 7. 客户端示例（pseudo-JS）

```js
const ws = new WebSocket(`ws://localhost:8001/ws/${execId}?token=${jwt}`);
ws.onopen = () => {
  ws.send(JSON.stringify({
    method: "subscribe_graph_execution",
    data: { graph_exec_id: execId },
  }));
};
ws.onmessage = (evt) => {
  const msg = JSON.parse(evt.data);
  switch (msg.method) {
    case "node_execution_event": updateNode(msg.data); break;
    case "graph_execution_event": updateGraph(msg.data); break;
    case "heartbeat": /* keep-alive */ break;
    case "error": console.error(msg.error); break;
  }
};
```

## 8. 边界条件

- **关闭竞态**：客户端关 WS 与 Redis 推消息可能并发；`conn_manager.py` 内部用 `asyncio.Lock` 保护订阅集合，并捕获 `WebSocketDisconnect` / `RuntimeError`。
- **Redis Cluster 模式**：必须用 Sharded PubSub（`SSUBSCRIBE` / `SPUBLISH`）；普通 `SUBSCRIBE` 会因为 channel 哈希到非本节点而收不到消息。
- **消息大小**：`event_bus.py` 对超大 payload 做 truncation，防止 Redis 连接崩溃。
- **频率**：高频 node 更新（如循环里 1k 次迭代）建议 ExecutionManager 侧聚合后再发；前端按 channel 节流渲染。

## 9. 测试

- `backend/api/conn_manager_test.py` —— ConnectionManager 单元；
- `backend/api/conn_manager_integration_test.py` —— 真 Redis 集群测试；
- `backend/api/ws_api_test.py` —— 启动一个 FastAPI test client，模拟订阅 & 消息推送。

## 10. 容器部署

`docker-compose.platform.yml: websocket_server`：

```yaml
command: ["ws"]
ports:
  - "8001:8001"
depends_on:
  - db / redis-0 / migrate / database_manager (healthy/completed)
```

水平扩容：多副本 + 前置 LB（如果用 sticky session 可以减少订阅迁移；不强制）。
