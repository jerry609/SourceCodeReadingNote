# Base Client：`AsyncAPIClient` 的核心能力

本节聚焦 SDK 的“发动机”：`src/tinker/_base_client.py`。

---

## 1. RequestOptions → httpx.Request

Resource 层最终都会调用到 `AsyncAPIClient.get/post/patch/...`，这些方法会：
- 拼接 `base_url + path`
- 合并 `headers/query/body/timeout/max_retries/...`
- 构造 `FinalRequestOptions`，再进入 `request()` 管线

关键入口：`AsyncAPIClient.request()`：`src/tinker/_base_client.py:895`

---

## 2. Request-level Retry Loop

`request()` 内部自带“单请求重试”：
- `max_retries = options.get_max_retries(self.max_retries)`
- 每次循环：
  1) `_build_request(options, retries_taken=...)`
  2) `httpx.AsyncClient.send(...)`
  3) 对 timeout / connection error / 4xx/5xx 分别处理

触发重试的典型条件：
- `httpx.TimeoutException`
- 其他 Exception（连接错误等）
- `httpx.HTTPStatusError` 且 `_should_retry(response)` 为真（常见：429/5xx）

backoff 逻辑集中在 `_sleep_for_retry()` 与 `_calculate_retry_timeout()`。

---

## 3. 幂等键（Idempotency Key）

为了让非 GET 请求在重试时具备幂等性，`request()` 会：
- 对非 GET 自动生成 idempotency key
- 在重试之间复用同一个 key

SDK 将 header 名称保存在 `self._idempotency_header`（由 `AsyncTinker` 在初始化后设置）。

---

## 4. 响应解析：从 httpx.Response 到 typed model

`request()` 成功后会进入 `_process_response()`：`src/tinker/_base_client.py:1040`

解析有几种分支：
- `cast_to == httpx.Response`：直接返回 raw response
- `RAW_RESPONSE_HEADER`：返回 `AsyncAPIResponse`（允许用户手动读取/解析）
- 默认：`AsyncAPIResponse.parse()` → `validate_type(...)` → 返回 typed model

这也是 `_strict_response_validation` 生效的位置：当服务端返回结构与预期不符时，严格模式会更早暴露问题。

---

## 5. 分页（Paginator）

`_base_client.py` 里还实现了通用分页抽象：
- `BasePage` / `AsyncPaginator`
- `PageInfo` 用于生成下一页请求参数

即使当前资源未大量使用分页，这套基础设施也解释了项目的整体代码生成/框架化风格。

