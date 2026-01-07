# 快速建立心智模型 (Reading Guide)

这份阅读指南的目标：用最短路径读懂 **Tinker Python SDK** 的“请求链路、同步封装、可靠性与 CLI 设计”。

---

## 0. 项目身份（30 秒）

一句话：**Tinker 是一个“客户端 SDK”** —— 它把一组 HTTP API（训练/采样/模型/权重/Telemetry…）包装成类型安全的 Python 接口，并额外提供：
- **同步调用体验**（后台线程 + asyncio event loop）
- **高层并发与重试**（连接限制 + progress timeout）
- **CLI**（Click + 懒加载）

你先回答 3 个问题就能开始读：
1) **请求是怎么发出去的？**（`resources/*` → `AsyncAPIClient.request()`）
2) **同步 API 为什么能并发？**（`InternalClientHolder`：后台 event loop + `run_coroutine_threadsafe`）
3) **重试在哪里发生？**（HTTP request-level 重试 + `RetryHandler` 高层重试）

---

## 1. 架构骨架（5 分钟）

先扫目录，建立“模块 → 职责”的映射：

- `src/tinker/_client.py`：顶层 client（`AsyncTinker`），负责鉴权 header、默认 base_url、错误映射（`_make_status_error`）。
- `src/tinker/_base_client.py`：核心引擎（`AsyncAPIClient`），负责 request 构造、重试、stream、响应解析。
- `src/tinker/resources/`：资源分组（`service/training/models/weights/sampling/futures/telemetry`），每个方法对应一个 endpoint。
- `src/tinker/types/`：数据结构与参数类型（多为生成代码）。
- `src/tinker/lib/public_interfaces/`：对外的同步接口（`ServiceClient/TrainingClient/SamplingClient/...`）。
- `src/tinker/lib/internal_client_holder.py`：同步接口背后的后台运行时（session、心跳、并发限制、telemetry）。
- `src/tinker/lib/retry_handler.py`：高层重试与进度跟踪（适合“发很多请求”的场景）。
- `src/tinker/cli/`：CLI（`LazyGroup` 懒加载 + 统一错误处理）。
- `tests/mock_api_server.py`：FastAPI mock server，用于端到端测试 SDK 行为。

---

## 2. 追数据流（10 分钟）

把 SDK 当成一条“函数管道”来读：

```
User code
  └─ client.<resource>.<method>(...)                    (resources/*.py)
       └─ AsyncAPIClient.<http_method>(path, ...)       (src/tinker/_base_client.py)
            └─ AsyncAPIClient.request(cast_to, options) (src/tinker/_base_client.py:895)
                 ├─ build httpx.Request + send
                 ├─ retry loop (max_retries)
                 └─ AsyncAPIResponse.parse() -> typed result
```

同步封装（Public Interfaces）的关键是：**把 async 调用塞进后台 event loop 运行**：

```
sync call
  └─ InternalClientHolder.run_coroutine_threadsafe(coro)
       └─ background thread loop.run_forever()
```

建议按下面顺序走读：
1) `src/tinker/_client.py:50`：`AsyncTinker` 初始化（API key/base_url）与错误映射。
2) `src/tinker/_base_client.py:895`：`request()` 的 retry loop 与 `_process_response()`。
3) `src/tinker/resources/service.py:21`：任选一个简单 endpoint（health / capabilities）看资源层怎么写。
4) `src/tinker/lib/internal_client_holder.py:107`：同步层的“后台运行时”如何创建 session、做心跳与并发控制。
5) `src/tinker/lib/retry_handler.py:90`：`RetryHandler.execute()` 怎么处理并发限制与 progress timeout。
6) `src/tinker/cli/__main__.py` + `src/tinker/cli/lazy_group.py:12`：CLI 懒加载怎么实现。

---

## 3. 锚点文件（80/20）

理解这些文件，你就能定位大多数问题：

- **client 入口**：`src/tinker/_client.py`
- **请求引擎**：`src/tinker/_base_client.py`
- **资源封装**：`src/tinker/resources/service.py`（最简单的起点）
- **同步运行时**：`src/tinker/lib/internal_client_holder.py`
- **高层重试**：`src/tinker/lib/retry_handler.py`
- **CLI**：`src/tinker/cli/__main__.py`，`src/tinker/cli/lazy_group.py`
- **测试基座**：`tests/mock_api_server.py`

---

## 4. 最小验证（读完就去做）

做一个最小闭环，验证你对链路的理解：
1) 跑 `pytest`，看 mock server 如何提供 endpoints。
2) 在本地写一个最小脚本：调用 `AsyncTinker().service.health_check()`（或同步 `ServiceClient().health_check()`）。
3) 临时把 `max_retries` 设为 0/1，对比错误行为（是否重试、报什么错）。

如果这些都能解释清楚，你就已经掌握了 SDK 的核心。

