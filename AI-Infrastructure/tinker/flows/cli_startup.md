# CLI 启动与懒加载（LazyGroup）

Tinker 的 CLI 设计目标非常明确：**`tinker --help` 也要很快**。

---

## 1) 入口：只 import Click

CLI 入口：`src/tinker/cli/__main__.py`
- 主命令 `main_cli` 用 `click.group(cls=LazyGroup, lazy_subcommands=...)` 注册子命令：`src/tinker/cli/__main__.py:19`
- 模块级尽量只 import `click` 与轻量工具，避免引入 SDK/网络/富格式输出等重依赖

---

## 2) LazyGroup：按需 import 子命令

关键实现：`src/tinker/cli/lazy_group.py:12`

`LazyGroup` 的核心是：把 `cmd_name -> "module.path:cli"` 的映射保存起来，只有当用户真的执行该子命令时才 import。

行为要点：
- `list_commands()`：把静态命令 + lazy 命令合并展示（保证 help 能列出命令）
- `get_command()`：命中 lazy 命令后，动态 import module 并取出入口函数

这能把“复杂命令需要的重依赖”推迟到真正执行时才加载，从而改善启动时间。

---

## 3) 统一错误处理：把 SDK 异常变成友好提示

CLI 的用户体验通常由“错误提示”决定：
- `src/tinker/cli/client.py` 提供 `handle_api_errors` 装饰器，捕获 `AuthenticationError/NotFoundError/RateLimitError/...` 并转成 `TinkerCliError`
- 主入口捕获 `TinkerCliError` 并输出简洁信息：`src/tinker/cli/__main__.py`

读 CLI 时建议顺序：
1) `cli/__main__.py`（命令树、format 参数、异常出口）
2) `cli/lazy_group.py`（懒加载）
3) `cli/client.py`（SDK client 创建与错误映射）

