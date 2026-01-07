# CLI：快启动、懒加载与错误体验

Tinker CLI 代码在 `src/tinker/cli/`，核心设计目标是：
- **启动快**：`--help` 不应加载重依赖
- **模块化**：命令独立开发/扩展
- **错误友好**：SDK 异常转成用户可读提示

---

## 1. 命令树：Click + LazyGroup

入口：`src/tinker/cli/__main__.py`
- 使用 `click.group(cls=LazyGroup, lazy_subcommands=...)` 注册子命令
- `lazy_subcommands` 将命令映射到 `"module.path:cli"` 的入口

关键实现：`src/tinker/cli/lazy_group.py:12`

详解见：[`AI-Infrastructure/tinker/flows/cli_startup.md`](../flows/cli_startup.md)

---

## 2. 统一 client 创建与错误映射

`src/tinker/cli/client.py` 做了两件重要事：

1) **client 创建**：`create_rest_client()`
- Lazy import `from tinker import ServiceClient`（避免启动时慢）
- 捕获常见异常，给出可操作的提示（缺 API key / 网络错误等）

2) **错误处理装饰器**：`handle_api_errors`
- 把 `AuthenticationError/NotFoundError/RateLimitError/...` 映射成 `TinkerCliError`
- `__main__.py` 统一捕获 `TinkerCliError` 并输出

这让命令实现可以专注业务逻辑，而不必每个命令都手写一套 try/except。

