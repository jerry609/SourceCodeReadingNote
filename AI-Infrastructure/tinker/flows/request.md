# 请求生命周期：构造 → 发送 → 重试 → 解析

这篇笔记回答一个核心问题：**从 `client.service.health_check()` 到拿到一个 typed response，中间发生了什么？**

---

## 1) Client 初始化：鉴权与默认配置

入口是 `AsyncTinker`：
- API key 从 `TINKER_API_KEY` 读取（未提供则报错），并要求以 `tml-` 前缀开头：`src/tinker/_client.py:54`
- base_url 从 `TINKER_BASE_URL` 读取（未提供则使用默认 URL）：`src/tinker/_client.py:54`
- 鉴权 header 固定为 `X-API-Key`：`src/tinker/_client.py:167`

---

## 2) Resource 层：把 endpoint 变成方法

以 `service.health_check()` 为例：
- `AsyncServiceResource.health_check()` 只是组装参数并调用 `_get("/api/v1/healthz", ...)`：`src/tinker/resources/service.py:44`
- `_get` 本质上是 `AsyncAPIClient.get()` 的别名：`src/tinker/_resource.py`

这里的关键点是：resource 方法会明确指定 `cast_to=...`，决定响应要解析成哪个类型。

---

## 3) Base Client：统一 request 管线

核心入口：`AsyncAPIClient.request()`：`src/tinker/_base_client.py:895`

请求处理流程大致是：

1. **准备阶段**
   - 缓存 platform 信息（可能涉及阻塞 IO，因此先在 async 上下文里做）
   - 复制 options，避免重试过程中被外部 mutation
   - 为非 GET 请求生成并复用 idempotency key（写请求幂等）

2. **重试循环**
   - `max_retries = options.get_max_retries(self.max_retries)`
   - 构造 `httpx.Request`，调用 `httpx.AsyncClient.send(...)`
   - 捕获 `TimeoutException` / 其他异常：如果还有重试次数，就 backoff 后继续；否则抛 `APITimeoutError`/`APIConnectionError`
   - `response.raise_for_status()` 触发 4xx/5xx：
     - 如果 `_should_retry(response)` 且仍有重试次数：关闭 response，sleep backoff，再试
     - 否则映射成 SDK 自己的 `APIStatusError` 子类（见下一节）

3. **响应解析**
   - `_process_response()` 会构造 `AsyncAPIResponse`，再 `parse()` 得到 typed model：`src/tinker/_base_client.py:1040`

---

## 4) 错误映射：从 HTTP status 到 SDK 异常

`AsyncTinker._make_status_error()` 负责把 status code 映射为更具体的异常类型：`src/tinker/_client.py:235`

常见映射：
- 400 → `BadRequestError`
- 401 → `AuthenticationError`
- 403 → `PermissionDeniedError`
- 404 → `NotFoundError`
- 429 → `RateLimitError`
- 5xx → `InternalServerError`

这让上层（包括 CLI）可以做精细化的错误提示与处理。

---

## 5) 多层重试：request-level / holder-level / high-level

在代码里你会看到不止一层“重试”，它们解决的问题粒度不同：

1) **HTTP request-level 重试**（`AsyncAPIClient.request()` 内部）
- 解决“单次请求”临时网络抖动/5xx/429 等问题
- 粒度小：每次只重试一个 HTTP 请求

2) **Holder-level 重试**（`InternalClientHolder.execute_with_retries()`）
- 解决“同步封装”场景下的一次业务调用可能触发的短暂失败（带上 Telemetry 记录）
- 有最大等待窗口（例如 5 分钟），并使用指数退避

3) **High-level `RetryHandler`**（`src/tinker/lib/retry_handler.py:90`）
- 解决“发很多请求/长时间并发”时的整体推进问题：连接并发限制、progress timeout、错误分类
- 典型用法：`SamplingClient` 用它包裹关键异步操作：`src/tinker/lib/public_interfaces/sampling_client.py:330`

读源码时建议先把它们的边界想清楚：**哪个层负责“单请求可靠性”，哪个层负责“业务调用可靠性”，哪个层负责“批量并发推进”**。
