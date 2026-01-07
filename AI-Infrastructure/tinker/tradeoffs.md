# 权衡取舍 (Tradeoffs)

## 1) Async-first + Sync wrapper：易用性 vs 复杂度

**选择**：核心 SDK 用 async（`AsyncTinker`），再通过 `InternalClientHolder` 提供同步接口。

**好处**：
- async 语义清晰，适合高并发/IO 密集
- 同步用户也能享受并发（后台 event loop），而不是“串行阻塞”

**代价**：
- 需要维护后台线程与 event loop 生命周期
- 调试更复杂（线程/协程交织）

参考：`src/tinker/_client.py`，`src/tinker/lib/internal_client_holder.py`

---

## 2) 多层重试：鲁棒性 vs 可解释性

可见的重试层包括：
- request-level：`AsyncAPIClient.request()`（单 HTTP 请求）
- holder-level：`InternalClientHolder.execute_with_retries()`（一次业务调用）
- high-level：`RetryHandler`（并发推进 + 无进度保护）

**好处**：对网络抖动、限流、服务端短暂故障更耐受。

**代价**：用户可能难以判断“为什么这么久还没失败/为什么重试这么多次”，需要更强的日志/telemetry 支撑。

---

## 3) `_strict_response_validation`：早失败 vs 兼容性

同步入口 `ServiceClient` 默认启用严格响应校验：`src/tinker/lib/public_interfaces/service_client.py`

**好处**：服务端返回 schema 偏差会被快速发现，避免“silent wrong data”。

**代价**：服务端如果灰度发布/字段变更，严格模式可能导致客户端更容易被破坏，需要配合更稳定的 API 演进策略。

---

## 4) CLI 依赖（rich 等）：体验 vs 启动性能

**选择**：用 Click + rich 提升交互体验，但通过 LazyGroup 把重依赖推迟到命令执行时加载。

**好处**：输出更友好，help 更清晰。

**代价**：命令实现需要遵守“不要在模块级 import 重依赖”的约束，否则会破坏启动性能。

参考：`src/tinker/cli/lazy_group.py`

---

## 5) 生成文档产物入库：可读性 vs PR 噪音

`scripts/generate_docs.py` 会生成 `docs/api/*.md` 并建议将生成产物提交进仓库。

**好处**：用户开箱即读 docs。

**代价**：API 变更会带来较大生成 diff，需要 CI/格式化保证稳定输出。

