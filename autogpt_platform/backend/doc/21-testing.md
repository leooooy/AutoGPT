# 21 测试策略

## 1. 测试栈

| 库                          | 作用                                                |
| --------------------------- | --------------------------------------------------- |
| `pytest ^8.4`               | 测试运行器                                          |
| `pytest-asyncio ^1.1`       | async 测试                                          |
| `pytest-mock ^3.15`         | `mocker` fixture                                    |
| `pytest-snapshot ^0.9`      | 快照测试（API JSON 响应）                           |
| `pytest-cov ^7.1`           | 覆盖率                                              |
| `pytest-watcher ^0.6`       | 文件改动自动重跑（开发用）                          |
| `faker`                     | 测试数据                                            |
| `httpx`                     | 同步 / 异步 HTTP（FastAPI TestClient 也基于此）     |

`pyproject.toml: [tool.pytest.ini_options]`：

- `asyncio_mode = "auto"` —— 默认全部 async；
- `asyncio_default_fixture_loop_scope = "session"` —— 共用一个 event loop；
- `addopts = "-p no:syrupy"` —— 与 pytest-snapshot 冲突；
- markers：
  - `supplementary` —— 维持覆盖率但已被集成测试取代；
  - `integration` —— 端到端，需要真 DB；
  - `slow` —— 用 docker stack 启动的，CI 可跳。

## 2. 文件布局

测试文件**与源码同目录**，命名 `xxx_test.py`：

```
backend/api/features/store/
├── routes.py
├── routes_test.py
├── db.py
├── db_test.py
├── ...
```

snapshot 文件位于子目录 `snapshots/`，与测试同级。

## 3. 跑测试

```bash
poetry run test                                # 全部
poetry run pytest path/to/file_test.py          # 单文件
poetry run pytest path/to/file_test.py::test_x  # 单函数
poetry run pytest -m "not slow"                 # 跳过 slow
poetry run pytest -k "credit"                   # 模式过滤
poetry run pytest --cov=backend                 # 覆盖率
```

`poetry run test` 等价于 `scripts/run_tests:test`，会：
1. 起一个 docker postgres（独立 schema）；
2. `prisma migrate deploy`；
3. 跑 `pytest`；
4. 退出时清理。

## 4. 鉴权 mock

主 API 路由用 `Security(get_jwt_payload)` 作根鉴权依赖。测试有两种思路：

### 4.1 完全跳过

`backend/api/conftest.py` 在测试启动时把 `ENABLE_AUTH=false`，此时 `get_user_id()` 直接返回 `DEFAULT_USER_ID`。**测试如果不关心具体 user_id**，什么都不用做。

### 4.2 mock 出指定 user

提供两个 fixture：

```python
mock_jwt_user   # user_id = "test-user-id"
mock_jwt_admin  # user_id = "admin-user-id"
```

用法：

```python
import fastapi
from fastapi.testclient import TestClient
import pytest
from backend.api.features.myroute import router

app = fastapi.FastAPI()
app.include_router(router)
client = TestClient(app)

@pytest.fixture(autouse=True)
def setup_app_auth(mock_jwt_user):
    from autogpt_libs.auth.jwt_utils import get_jwt_payload
    app.dependency_overrides[get_jwt_payload] = mock_jwt_user["get_jwt_payload"]
    yield
    app.dependency_overrides.clear()
```

管理员路由换 `mock_jwt_admin`。`target_user_id` 用于 admin 操作他人数据的场景。

## 5. Snapshot 测试

适合 API 响应稳定的端点：

```python
import json
from pytest_snapshot.plugin import Snapshot

def test_endpoint(snapshot: Snapshot):
    resp = client.get("/api/some-endpoint")
    assert resp.status_code == 200
    data = resp.json()
    data.pop("created_at", None)        # 去掉动态字段
    data.pop("id", None)
    snapshot.snapshot_dir = "snapshots"
    snapshot.assert_match(
        json.dumps(data, indent=2, sort_keys=True),
        "endpoint_response",
    )
```

更新快照：

```bash
poetry run pytest path/to/test.py --snapshot-update
git diff path/to/snapshots/  # 必看 diff
```

⚠️ 注意：snapshot diff 不要"无脑 commit"。每次更新都意味着 API 行为变了，需要确认是否预期。

## 6. Mock 边界

按 `backend/AGENTS.md` 的规则：

- **mock 在使用处，不在定义处**。例如 `backend/api/features/store/routes.py` 里 `from backend.data.store import get_agent`，那么测试要 mock `backend.api.features.store.routes.get_agent`，**不**是 `backend.data.store.get_agent`。
- 重构后路径变了 → 找 mock 路径同步更新。
- async function 用 `from unittest.mock import AsyncMock`。

## 7. 集成 vs 单元

| 类型 | 触发                           | 速度 | 例子                                          |
| ---- | ------------------------------ | ---- | --------------------------------------------- |
| 单元 | 默认                           | 快   | 业务逻辑、纯函数、Pydantic 校验               |
| 集成 | `@pytest.mark.integration`     | 中   | 路由 + 真 Postgres + 真 Redis                  |
| 慢   | `@pytest.mark.slow`            | 慢   | 起完整 docker stack（执行端到端）             |

CI 默认跳过 `slow`；本地 review 时手动跑：

```bash
poetry run pytest -m slow
```

## 8. 测试 Block

`backend/blocks/test/test_block.py` —— 通用扫描，对所有 Block：

- 检查注册（id 唯一、UUID 合法）；
- 检查 input/output schema 合法性；
- 如果 Block 类有 `test_input` 和 `test_output` 字段，自动跑一次 run() 比对结果。

跑某个 Block：

```bash
poetry run pytest 'backend/blocks/test/test_block.py::test_available_blocks[GetCurrentTimeBlock]' -xvs
```

## 9. TDD 工作流（仓库约定）

修 bug 或加新行为时：

1. **先写一个失败的测试**，标 `@pytest.mark.xfail(reason="...")`：

   ```python
   @pytest.mark.xfail(reason="bug #1234: empty input crashes widget")
   def test_widget_handles_empty_input():
       assert widget.process("") == Widget.EMPTY_RESULT
   ```

2. 跑它，确认是 XFAIL（"按预期失败"）；
3. 写最小代码让它通过；
4. **删掉 `xfail` 标记**，再跑一次，确认绿灯；
5. 跑全套，没破其他东西。

> 这是 backend `AGENTS.md` 强烈建议的范式 —— 每次 fix 留下 regression 防线。

## 10. 数据库测试隔离

- `backend/conftest.py` / `backend/api/conftest.py` 里有数据库准备 / 清理逻辑；
- 集成测试通常在测试结束后回滚事务；
- 真要跑迁移变化 → 用独立 schema + `setup_method` / `teardown_method`。

## 11. 常用 fixture

| fixture                | 内容                                       |
| ---------------------- | ------------------------------------------ |
| `mock_jwt_user`        | 普通用户 JWT mock                          |
| `mock_jwt_admin`       | 管理员 JWT mock                            |
| `test_user_id`         | 字符串 ID                                  |
| `admin_user_id`        | 字符串 ID                                  |
| `target_user_id`       | 用于 admin 操作他人数据                    |
| `configured_snapshot`  | 已设置 `snapshot_dir` 的 Snapshot 实例     |
| `mocker`               | pytest-mock                                |
| `tmp_path`             | 临时目录                                   |

## 12. 测试新路由的最佳实践

```python
# backend/api/features/widget/routes_test.py
import json
import fastapi
from fastapi.testclient import TestClient
import pytest
from pytest_snapshot.plugin import Snapshot
from backend.api.features.widget.routes import router

app = fastapi.FastAPI()
app.include_router(router)
client = TestClient(app)


@pytest.fixture(autouse=True)
def setup_app_auth(mock_jwt_user):
    from autogpt_libs.auth.jwt_utils import get_jwt_payload
    app.dependency_overrides[get_jwt_payload] = mock_jwt_user["get_jwt_payload"]
    yield
    app.dependency_overrides.clear()


def test_create_widget_happy_path(snapshot: Snapshot, mocker):
    mocker.patch(
        "backend.api.features.widget.routes.create_widget",
        return_value={"id": "w-001", "name": "Demo"},
    )
    resp = client.post("/widgets", json={"name": "Demo"})
    assert resp.status_code == 201
    snapshot.snapshot_dir = "snapshots"
    snapshot.assert_match(
        json.dumps(resp.json(), indent=2, sort_keys=True),
        "create_widget_happy",
    )


def test_create_widget_invalid_input():
    resp = client.post("/widgets", json={"name": ""})
    assert resp.status_code == 422
```

## 13. CI

仓库的 GitHub Actions：

- 每个 PR 跑 lint / type-check / 单元 + 集成；
- snapshot 文件被提交到仓库，CI 比对；
- `slow` 跳过；
- 失败发 PR 评论。

## 14. 常见坑

| 坑                                                  | 解决                                                                   |
| --------------------------------------------------- | ---------------------------------------------------------------------- |
| `RuntimeError: Event loop is closed`                | `asyncio_default_fixture_loop_scope = "session"` 已开；自定义 fixture 不要重置 loop |
| Mock 路径错（"why is the real call still running"）  | mock 在 *使用* 处                                                      |
| Snapshot 一直 diff 时间戳                            | 测试里 pop 掉 `created_at` / `updated_at` / `id`                        |
| AsyncMock 返回 `<coroutine ...>`                     | `AsyncMock(return_value=...)` 而不是 `Mock(return_value=...)`          |
| 集成测试影响后续测试                                  | 用事务回滚或显式删除                                                   |
| 路由测试 401                                          | 没 setup auth fixture                                                  |
