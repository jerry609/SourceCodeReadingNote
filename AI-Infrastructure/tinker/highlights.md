# Tinker 惊艳之处 (Highlights)

## 1) CLI 懒加载：`tinker --help` 也要快

- `LazyGroup` 把子命令映射为 import path，运行时按需加载：`src/tinker/cli/lazy_group.py:12`
- 避免在 CLI 启动时 import SDK/网络/富输出等重依赖

---

## 2) 同步封装不“假装同步”：直接上后台 event loop

- `InternalClientHolder` 用后台线程维护 event loop，并用 `run_coroutine_threadsafe` 把 async 调用暴露给同步用户
- 不是“同步阻塞网络调用”的简单包装，而是可并发、可限流、可观测的运行时

参考：`src/tinker/lib/internal_client_holder.py`

---

## 3) Sampling 限流做到了“三维”

对采样这种高频/高带宽调用，`InternalClientHolder` 同时限制：
- 并发请求数（semaphore）
- backoff 近期更严格的限流（避免雪崩）
- 估算字节数的限流（BytesSemaphore）

参考：`src/tinker/lib/internal_client_holder.py:150`

---

## 4) RetryHandler 的“无进度保护”

`RetryHandler` 不只做重试，还会监控整体 progress：
- 长时间无进度 → cancel → 抛出带诊断信息的错误
- 对“高并发+连接资源受限”的场景更友好

参考：`src/tinker/lib/retry_handler.py:90`

---

## 5) 端到端测试基座：Mock API Server

`tests/mock_api_server.py` 提供完整 endpoints 的 mock，实现端到端行为测试（包括类型解析与错误映射）。

