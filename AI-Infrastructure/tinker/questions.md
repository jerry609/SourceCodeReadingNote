# 问题与解答 (Q&A)

## Q1：API Key 和 Base URL 从哪里配置？

**A**：默认从环境变量读取：
- `TINKER_API_KEY`：必须提供，否则初始化 client 会失败（并要求 `tml-` 前缀）
- `TINKER_BASE_URL`：可选，不提供则使用默认 URL

参考：`src/tinker/_client.py:54`

---

## Q2：为什么要校验 API key 的 `tml-` 前缀？

**A**：这是一个“早失败”策略：在 client 初始化阶段就能发现明显的配置错误（例如复制错 key 或拿了别的系统的 token）。

参考：`src/tinker/_client.py:54`

---

## Q3：怎么关闭重试？

**A**：需要分清你要关的是哪层重试：
- 关闭 **request-level 重试**：在调用 resource 方法时传 `max_retries=0`（或创建 client 时把默认 `max_retries` 设为 0）
- 关闭 **holder-level/RetryHandler 重试**：需要修改/绕过 `InternalClientHolder.execute_with_retries()` 或 `RetryHandler(enable_retry_logic=False)`

参考：`src/tinker/_base_client.py:895`，`src/tinker/lib/internal_client_holder.py:317`，`src/tinker/lib/retry_handler.py:39`

---

## Q4：`with_raw_response` / `with_streaming_response` 是干什么的？

**A**：
- `with_raw_response`：返回 `httpx.Response`/`AsyncAPIResponse`，你可以自己读取 headers/body，适合调试或需要低层信息的场景
- `with_streaming_response`：用于不想一次性把响应读入内存的场景（例如下载大文件/二进制内容）

参考：`src/tinker/_client.py:152`

---

## Q5：为什么 SDK 里会出现多层重试？

**A**：它们解决的“失败范围”不同：
- request-level：单个 HTTP 请求的瞬时失败
- holder-level：一次业务调用（同步封装）在短时间内反复尝试推进
- RetryHandler：大并发/长流程的整体推进与“无进度”保护

参见：`AI-Infrastructure/tinker/flows/request.md`

---

## Q6：同步接口是怎么做到并发的？

**A**：同步接口把 async SDK 放到后台线程的 event loop 中运行：
1) 后台线程 `loop.run_forever()`
2) 同步方法通过 `run_coroutine_threadsafe` 提交协程并拿到 future

参考：`src/tinker/lib/internal_client_holder.py`

---

## Q7：SDK 为什么要维护 session 与心跳？

**A**：对于“带资源分配/队列”的服务端，session 能承载：
- 资源分配与生命周期
- Telemetry 关联（让服务端/SDK 观测更容易）
- 心跳维持与异常告警（连接不稳定/服务端不可用时可提示用户）

参考：`src/tinker/lib/internal_client_holder.py:186`

---

## Q8：文档是怎么生成的？为什么要把生成产物提交进仓库？

**A**：文档由脚本扫描模块、调用 `pydoc-markdown` 生成 Markdown，并产出 `docs/api/_meta.json` 作为导航：
- 生成脚本：`scripts/generate_docs.py:85`
- 生成目录：`docs/api/`

把生成产物提交进仓库的好处是：用户不需要本地生成即可阅读；缺点是：改动 API 时可能产生较大 diff。

