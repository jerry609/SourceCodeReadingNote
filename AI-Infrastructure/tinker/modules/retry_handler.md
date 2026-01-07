# RetryHandler：高层重试与“无进度”保护

`RetryHandler` 不是单次 HTTP 请求的 retry（那在 `AsyncAPIClient.request()` 里），它更像是给“高层操作”提供一个通用的可靠性包装：
- 限制并发连接（避免把 httpx 连接池打爆导致整体停滞）
- 在失败时指数退避 + 抖动
- 记录 telemetry
- 如果长时间“无进度”，主动失败（避免无限卡住）

源码：`src/tinker/lib/retry_handler.py:90`

---

## 1. 并发限制：Semaphore

`RetryConfig.max_connections` 控制并发上限：
- `RetryHandler.execute()` 一开始会 `async with self._semaphore` 获取配额
- 没有这个限制时，大量并发请求会抢占 httpx 连接，导致整体 progress 变差（作者在注释里明确提到这一点）

---

## 2. 无进度保护：progress timeout

`execute()` 会启动一个后台任务 `_check_progress`：
- 计算 `deadline = last_global_progress + progress_timeout`
- 超时则 cancel 当前 task
- 被 cancel 后会转成 `APIConnectionError`，错误信息是 “No progress made ...”

关键点：这不是单请求超时，而是“**系统层面**已经很久没有任何进展”的保护。

---

## 3. 重试判定：异常分类

`_should_retry()` 的判定规则：
- `RetryConfig.retryable_exceptions`（默认含 timeout/connection/retryable_exception）
- 或者 `APIStatusError` 且 status code 可重试（408/409/429/5xx）

对应 helper：`is_retryable_status_code()`：`src/tinker/lib/retry_handler.py:34`

---

## 4. Backoff：指数退避 + jitter

`_calculate_retry_delay(attempt)`：
- `delay = min(base * 2^attempt, max)`
- `jitter = delay * jitter_factor * (2 * rand - 1)`
- 最终 delay clamp 到 `[0, retry_delay_max]`

这是一种典型的“指数退避 + 抖动”策略，用于避免大量客户端同时重试造成的同步风暴（thundering herd）。

---

## 5. 在哪里用到？

一个典型用法是 `SamplingClient`：
- 把核心 async 操作包在 `self.retry_handler.execute(...)` 里
- 当底层 API 短暂 429/5xx/超时波动时，能够自动推进而不是立刻失败

参见：`src/tinker/lib/public_interfaces/sampling_client.py:330`

