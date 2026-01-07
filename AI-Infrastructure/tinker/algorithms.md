# 关键算法 (Key Algorithms)

本节记录 Tinker SDK 中几个“值得单独抽出来理解”的小算法/机制。

---

## 1) HTTP request-level 重试（`AsyncAPIClient.request`）

位置：`src/tinker/_base_client.py:895`

核心是一个 `for retries_taken in range(max_retries + 1)` 循环：

```text
options = copy(input_options)
request = build_request(options)
resp = httpx.send(request)
if timeout/connection error:
  if remaining_retries > 0: sleep(backoff); continue
  else: raise
if resp.status is 4xx/5xx:
  if remaining_retries > 0 && should_retry(resp): sleep(backoff); continue
  else: raise mapped APIStatusError
return parse(resp)
```

注意点：
- 非 GET 会自动生成并复用 idempotency key（让重试更安全）
- 解析阶段使用 `AsyncAPIResponse.parse()` + typed validation

---

## 2) holder-level 指数退避（`InternalClientHolder.execute_with_retries`）

位置：`src/tinker/lib/internal_client_holder.py:317`

特点：
- 有最大等待窗口（例如 5 分钟）
- 指数退避：`time_to_wait = min(2**attempt_count, 30)`
- 每次异常都会写 telemetry（包括异常栈、是否 retryable、elapsed 等）

这更像“业务调用级”重试，而不是“单请求级”重试。

---

## 3) RetryHandler：指数退避 + jitter + 无进度保护

位置：`src/tinker/lib/retry_handler.py:90`

### 3.1 指数退避 + jitter

```text
delay = min(base * 2^attempt, max)
jitter = delay * jitter_factor * (2*rand - 1)
sleep(clamp(delay + jitter, 0..max))
```

意义：
- 指数退避：失败越多等待越久，避免过载
- jitter：避免大量客户端同步重试导致“重试风暴”

### 3.2 无进度保护（progress timeout）

用 `last_global_progress + progress_timeout` 作为 deadline：
- 超过 deadline → cancel 当前任务
- 转换成带诊断信息的 `APIConnectionError`

这是“系统整体”层面的兜底，避免请求队列卡死时无限等待。

---

## 4) SSEDecoder：按 chunk 边界解码事件

位置：`src/tinker/_streaming.py:189`

核心逻辑是 `_iter_chunks`：
- 累积 bytes
- 当 buffer 以 `\r\r` / `\n\n` / `\r\n\r\n` 结尾时切出一个 SSE chunk
- 对每个 chunk `splitlines()`，逐行 `decode(...)`，遇到空行输出一个完整 event

这类实现的价值在于：**网络分包不对齐 event 边界** 时仍能正确解析。

