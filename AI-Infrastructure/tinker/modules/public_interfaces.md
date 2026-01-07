# Public Interfaces：同步封装与后台运行时

Tinker SDK 不只提供 `AsyncTinker`，还提供更“产品化”的同步接口：`ServiceClient/TrainingClient/SamplingClient/RestClient`。

它们的共同点是：**把 async SDK 放进后台线程的 event loop 里跑**，从而让同步代码也能并发调用。

---

## 1. 入口：`ServiceClient`

`ServiceClient` 是同步世界的入口：`src/tinker/lib/public_interfaces/service_client.py:31`

初始化时会创建 `InternalClientHolder`：
- 注入默认 headers（含一些 SDK metadata）
- 开启 `_strict_response_validation=True`（让类型不一致尽早暴露）
- 创建并保存 session（服务端分配资源/用于 telemetry）

---

## 2. 核心：`InternalClientHolder`

`InternalClientHolder` 是同步封装的关键：`src/tinker/lib/internal_client_holder.py:107`

它做了几件“SDK 产品化”才会做的事：

### 2.1 后台 event loop（线程单例）
- `InternalClientHolderThreadSingleton` 在后台 thread 里 `loop.run_forever()`
- 同步接口通过 `run_coroutine_threadsafe(...)` 把协程提交到该 loop 执行

### 2.2 Async client 连接池
- `ClientConnectionPool` 按 `ClientConnectionPoolType` 维护多个 `AsyncTinker` 实例
- 用 `MAX_REQUESTS_PER_HTTPX_CLIENT` 做简单轮转（避免单 httpx client 使用过久导致的异常行为/连接问题）

### 2.3 会话与心跳
- 初始化时创建 session（并读取 `TINKER_TAGS` 等元信息）
- 启动后台心跳任务，周期性调用 `service.session_heartbeat(...)`，并在连续失败时告警

### 2.4 并发与速率控制（Sampling）
`sample_dispatch_rate_limit()` 通过多种 semaphore 限制：
- 并发请求数（全局）
- “backoff 近期”时更严格的限流
- 估算字节数的限流（`BytesSemaphore`）

### 2.5 holder-level 重试
`execute_with_retries()` 自带指数退避与最大等待窗口，并在每次异常时通过 telemetry 记录上下文：
- `is_retryable`（按 status code / exception 分类）
- `is_user_error`（用于区分用户输入错误 vs 系统错误）

---

## 3. 同步接口的“异步提交模式”

同步 API 通常长这样：

1) 定义 `_xxx_submit()`：
- 内部写一个 async closure
- 从 holder 获取 `AsyncTinker` client（`with holder.aclient(...) as client:`）
- 调用资源层方法（`client.service.get_server_capabilities()` 等）
- 通过 `holder.execute_with_retries(...)` 包一层业务重试

2) 对外提供同步方法：
- `return _xxx_submit().result()`

这一套让“业务重试/并发限制/telemetry”可以集中在 holder，而不是散落在每个 API 方法里。

