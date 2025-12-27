# JuiceFS 源码分析笔记

## 1. 项目概览 (Overview)
*   **类型**: 分布式 POSIX 文件系统 (Cloud-native Distributed File System)
*   **核心语言**: Go
*   **一句话描述**: 插件化元数据引擎 + 对象存储，实现高性能、无限扩展的 POSIX 文件系统。
*   **阅读目标**: 建立心智模型，理清数据流与元数据流，为 Rust 重写打下基础。

## 2. 核心架构 (Architecture)
JuiceFS 采用元数据与数据完全分离的架构：

```
┌─────────────────────────────────────────────────────────────┐
│                     JuiceFS Client                          │
├─────────────┬─────────────┬─────────────┬──────────────────┤
│   pkg/fuse  │   pkg/fs    │ pkg/gateway │     pkg/vfs      │
│  (FUSE 接口) │ (Hadoop 接口)│  (S3 网关)   │  (VFS 抽象层)   │
├─────────────┴─────────────┴─────────────┴──────────────────┤
│                        pkg/vfs                              │
│              (文件句柄管理、读写分发、状态恢复)               │
├────────────────────────────┬───────────────────────────────┤
│         pkg/meta           │         pkg/chunk              │
│    (元数据引擎抽象层)        │    (数据分块与缓存层)          │
├────────────────────────────┼───────────────────────────────┤
│  Redis │ SQL │ TiKV │ ... │    pkg/object (对象存储抽象)   │
├────────────────────────────┼───────────────────────────────┤
│     Metadata Storage       │      Object Storage           │
│  (Redis/MySQL/TiKV/...)    │   (S3/OSS/GCS/MinIO/...)      │
└────────────────────────────┴───────────────────────────────┘
```

### 核心概念
- **Metadata Engine**: 存储文件系统元数据（inode, dentry, xattr, locks 等）。支持 Redis, MySQL, TiKV 等。
- **Object Storage**: 存储文件内容（Chunk, Slice, Block）。支持 S3, GCS, OSS 等 50+ 后端。
- **JuiceFS Client**: 实现 FUSE 挂载、S3 接口、Hadoop 接口。

### 数据分层
```
文件 -> Chunk (64MB) -> Block (4MB) -> Page (64KB)
```

## 3. 关键路径 (Critical Paths)
- [Lookup 流程](flows/lookup.md): 目录项查找与元数据获取。
- [Write 流程](flows/write.md): 数据分块、上传对象存储、更新元数据。
- [Read 流程](flows/read.md): 获取 Slice 列表、从对象存储拉取、缓存。

## 4. 模块分析 (Modules)
- [pkg/meta](modules/pkg_meta.md): 元数据接口与后端实现 (Redis/SQL/TiKV)。
- [pkg/chunk](modules/pkg_chunk.md): 数据分块、缓存、Singleflight、与对象存储交互。
- [pkg/object](modules/pkg_object.md): 对象存储抽象层 (50+ 后端支持)。
- [pkg/vfs](modules/pkg_vfs.md): VFS 实现、文件句柄管理、热升级支持。

## 5. 重点类/结构体 (Key Structures)
| 结构体 | 位置 | 说明 |
|:-------|:-----|:-----|
| `Ino` (u64) | `pkg/meta/interface.go:116` | Inode 唯一标识 |
| `Attr` | `pkg/meta/interface.go:154` | 文件属性（Mode, Size, Uid, Gid, Time 等） |
| `Slice` | `pkg/meta/interface.go:314` | 对象存储中数据片段的描述 |
| `Entry` | `pkg/meta/interface.go:306` | 目录项（Inode + Name + Attr） |
| `Meta` | `pkg/meta/interface.go:372` | 元数据引擎接口（60+ 方法） |
| `ChunkStore` | `pkg/chunk/chunk.go:38` | 数据块存储接口 |
| `ObjectStorage` | `pkg/object/interface.go:77` | 对象存储接口 |
| `handle` | `pkg/vfs/handle.go:32` | 文件句柄（含 reader/writer） |

## 6. 疑问与解答 (Q&A)
见 [questions.md](questions.md) - 包含 10+ 个已解答的核心问题。

## 7. 惊艳之处 (Highlights)
见 [highlights.md](highlights.md) - 记录令人印象深刻的设计：
- Lua 脚本合并 Redis 请求 (50% 延迟优化)
- Singleflight 防止缓存击穿
- 插件化元数据引擎架构
- Page Pool 内存复用
- VFS 热升级状态恢复
- 数据压缩与 Range Request 结合

## 8. 关键算法 (Key Algorithms)
见 [algorithms.md](algorithms.md) - 核心算法详解：
- Singleflight (请求合并)
- Chunk 分块算法
- 二进制属性序列化
- LRU 缓存淘汰
- 预读算法 (Prefetch)
- Writeback 异步上传

## 9. 权衡取舍 (Tradeoffs)
见 [tradeoffs.md](tradeoffs.md) - 设计决策分析：
- 元数据与数据分离存储
- 块大小选择 (4MB)
- 同步写入 vs 异步写入
- 元数据接口的大小
- Redis Lua 脚本 vs 多次调用
- 客户端缓存一致性
- Go vs Rust

## 10. 思考与重构 (Reflections)
- **Rust 重写选型**:
    - 对象存储: `OpenDAL` (Apache 开源，50+ 后端)
    - FUSE: `fuse-backend-rs` (阿里开源，支持 virtio-fs)
    - 元数据: `sqlx`, `redis-rs`
    - 异步: `Tokio`
    - 缓存: `moka` (高性能并发 LRU)

## 8. 实战练习 (Exercises)
见 [exercises/TASKS.md](exercises/TASKS.md) - Rust 重写任务清单。

## 11. 面试题库 (Interview Questions)
见 [interview_questions.md](interview_questions.md) - 100 道面试题，覆盖架构、实现、算法、运维等维度。