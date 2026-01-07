# Tinker 源码分析笔记

## 1. 项目概览 (Overview)
*   **类型**: AI 平台/训练服务的 Python SDK（客户端库）
*   **核心语言**: Python
*   **一句话描述**: 提供一套 *typed + async-first* 的 API，封装 Tinker 服务端的训练/采样/模型/权重/Telemetry 等接口，并提供同步封装与 CLI。
*   **阅读目标**: 搞清楚一次请求如何构造/重试/解析；同步封装如何用后台 event loop 并发；CLI 如何做到“快启动 + 懒加载”。

快速阅读路线：[`AI-Infrastructure/tinker/reading-guide.md`](reading-guide.md)

## 2. 核心架构 (Architecture)

Tinker SDK 的核心是“**分层**”：
- **Resource 层**：每个模块负责一组 HTTP endpoints（`service/training/models/...`）。
- **Base Client 层**：统一处理 request 构建、重试、错误映射、响应解析。
- **Public Interface 层**：为同步代码提供友好 API（后台线程 + event loop + 并发控制）。
- **CLI 层**：Click + LazyGroup 懒加载，保证 `--help` 也很快。

```
┌───────────────────────────┐
│        User / CLI          │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│ Public Interfaces (sync)   │  src/tinker/lib/public_interfaces/*
│ + InternalClientHolder     │  src/tinker/lib/internal_client_holder.py
└─────────────┬─────────────┘
              │ (calls)
              ▼
┌───────────────────────────┐
│ AsyncTinker (AsyncAPIClient)│ src/tinker/_client.py:50
└─────────────┬─────────────┘
              │ (resource methods)
              ▼
┌───────────────────────────┐
│ Resources                 │  src/tinker/resources/*
└─────────────┬─────────────┘
              │ (httpx)
              ▼
┌───────────────────────────┐
│        Tinker API          │
└───────────────────────────┘
              ▲
              │
┌─────────────┴─────────────┐
│ Tests: Mock API server     │ tests/mock_api_server.py:27
└───────────────────────────┘
```

## 3. 关键路径 (Critical Paths)

- [请求生命周期（构造/重试/解析）](flows/request.md)
- [CLI 启动与懒加载](flows/cli_startup.md)

## 4. 模块分析 (Modules)
- [Base Client（请求/重试/解析）](modules/base_client.md)
- [Public Interfaces（同步封装/会话/并发）](modules/public_interfaces.md)
- [RetryHandler（高层重试与进度跟踪）](modules/retry_handler.md)
- [CLI（LazyGroup + 错误处理）](modules/cli.md)

## 5. 重点类/结构体 (Key Structures)

| 结构体/概念 | 位置 | 作用 |
|:--|:--|:--|
| `AsyncTinker` | `src/tinker/_client.py:50` | 顶层 async client：鉴权、默认 base_url、资源入口、错误映射 |
| `AsyncAPIClient.request()` | `src/tinker/_base_client.py:895` | 统一的 HTTP request 管线：重试、stream、响应解析 |
| `Async*Resource` | `src/tinker/resources/*.py` | endpoint 封装（调用 `_get/_post`，传入 `cast_to`） |
| `InternalClientHolder` | `src/tinker/lib/internal_client_holder.py:107` | 同步 API 的“后台 async 引擎”：线程 event loop + session + rate limit |
| `RetryHandler` | `src/tinker/lib/retry_handler.py:90` | 高层重试：连接并发限制 + progress timeout + backoff |
| `LazyGroup` | `src/tinker/cli/lazy_group.py:12` | Click 子命令懒加载，保持 CLI 启动速度 |
| `SSEDecoder` | `src/tinker/_streaming.py:189` | SSE 事件解码（通用实现，支持 chunk 边界） |

## 6. 疑问与解答 (Q&A)
见 [`AI-Infrastructure/tinker/questions.md`](questions.md)

## 7. 惊艳之处 (Highlights)
见 [`AI-Infrastructure/tinker/highlights.md`](highlights.md)

## 8. 关键算法 (Key Algorithms)
见 [`AI-Infrastructure/tinker/algorithms.md`](algorithms.md)

## 9. 权衡取舍 (Tradeoffs)
见 [`AI-Infrastructure/tinker/tradeoffs.md`](tradeoffs.md)

## 10. 实战练习 (Exercises)
见 [`AI-Infrastructure/tinker/exercises/TASKS.md`](exercises/TASKS.md)

