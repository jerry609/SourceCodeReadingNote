# pkg/vfs 模块分析

## 职责
`pkg/vfs` 是 FUSE 协议与 JuiceFS 内部逻辑的适配层。它实现了 FUSE 的所有文件系统操作，并维护了内核最关心的状态——**文件句柄 (File Handle)**。

## 核心结构体

### `VFS`
上帝对象，持有了 `Meta`, `ChunkStore` 以及句柄表。
```go
type VFS struct {
    Meta  meta.Meta
    Store chunk.ChunkStore
    handles map[Ino][]*handle // inode -> handles 映射
    // ...
}
```

### `handle`
代表一个打开的文件。
```go
type handle struct {
    inode  Ino
    fh     uint64
    reader FileReader // 指向 chunk 实现
    writer FileWriter // 指向 chunk 实现
    // ...
}
```

## 关键流程

### 打开文件 (Open)
1.  `vfs.Open` 接收 `inode` 和 `flags`。
2.  调用 `meta.Open` 检查权限。
3.  调用 `newFileHandle` 分配唯一的 `fh` (uint64)。
4.  根据 flags 初始化 `reader` 和 `writer` (连接到 Chunk Store)。
5.  返回 `fh` 给内核。

### 读写分发
- **Read**: `vfs.Read(fh, ...)` -> 查找 handle -> `handle.reader.Read` -> `chunk.ReadAt`
- **Write**: `vfs.Write(fh, ...)` -> 查找 handle -> `handle.writer.Write` -> `chunk.WriteAt`

### 状态恢复 (State Recovery)
JuiceFS 支持热升级。`vfs.go` 中的 `dumpAllHandles` 和 `loadAllHandles` 负责将当前打开的句柄状态序列化到磁盘（通常是 JSON），以便重启后恢复，保证业务不中断。

## Rust 重写思考

### 库选型：`fuse-backend-rs`
- 相比 `fuser`，阿里开源的 [fuse-backend-rs](https://github.com/dragonflyoss/fuse-backend-rs) 更适合生产级存储系统。
- 它支持多线程模型，且抽象了 `FileSystem` trait，方便未来扩展到 virtio-fs。

### 并发模型
- **DashMap**: 用于管理 `fh -> Arc<Handle>` 的映射，提供高性能并发读写。
- **RwLock**: 在 Handle 内部使用 `RwLock` 或 `Mutex` 保护读写偏移量等状态。

### 模块解耦
建议将 `VFS` 拆分为：
- **InodeManager**: 处理 `Lookup`, `GetAttr`, `SetAttr` 等元数据操作。
- **HandleManager**: 处理 `Open`, `Release`, `Read`, `Write` 等 IO 操作。
