# JuiceFS 源码分析笔记

## 1. 项目概览 (Overview)
*   **类型**: 分布式 POSIX 文件系统 (Cloud-native Distributed File System)
*   **核心语言**: Go
*   **一句话描述**: 插件化元数据引擎 + 对象存储，实现高性能、无限扩展的 POSIX 文件系统。
*   **阅读目标**: 建立心智模型，理清数据流与元数据流，为 Rust 重写打下基础。

## 2. 核心架构 (Architecture)
JuiceFS 采用元数据与数据完全分离的架构：
- **Metadata Engine**: 存储文件系统元数据（inode, dentry, xattr, locks 等）。支持 Redis, MySQL, TiKV 等。
- **Object Storage**: 存储文件内容（Chunk, Slice, Block）。支持 S3, GCS, OSS 等。
- **JuiceFS Client**: 实现 FUSE 挂载、S3 接口、Hadoop 接口。

## 3. 关键路径 (Critical Paths)
- [Lookup 流程](flows/lookup.md): 目录项查找与元数据获取。
- [Write 流程](flows/write.md): 数据分块、上传对象存储、更新元数据。
- [Read 流程](flows/read.md): 获取 Slice 列表、从对象存储拉取、缓存。

## 4. 模块分析 (Modules)
- [pkg/meta](modules/pkg_meta.md): 元数据接口与后端实现 (Redis/SQL)。
- [pkg/chunk](modules/pkg_chunk.md): 数据分块与 Slice 管理逻辑。
- [pkg/object](modules/pkg_object.md): 对象存储抽象层。
- [pkg/fs](modules/pkg_fs.md): FUSE 实现与 VFS 胶水层。

## 5. 重点类/结构体 (Key Structures)
- `Ino` (u64): Inode 唯一标识。
- `Attr`: 文件属性（Mode, Size, Uid, Gid, Mtime 等）。
- `Slice`: 对象存储中数据片段的描述。

## 6. 思考与重构 (Reflections)
- **Rust 重写选型**:
    - 对象存储: `OpenDAL`
    - FUSE: `fuse-backend-rs`
    - 元数据: `sqlx`, `redis-rs`
    - 异步: `Tokio`