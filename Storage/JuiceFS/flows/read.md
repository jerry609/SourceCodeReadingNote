# JuiceFS 读取流程 (Read Path)

## 概述
JuiceFS 的读取逻辑旨在通过多级缓存（内存、磁盘）和积极的预读（Prefetch）来屏蔽对象存储的高延迟。

## 核心流程图

```mermaid
graph TD
    A[Client: fs.Read] --> B[vfs.Read]
    B --> C[chunk.rSlice.ReadAt]
    
    C --> D{本地磁盘缓存?}
    D -- "命中 (Hit)" --> E[从本地磁盘读取 Block]
    E --> F[返回数据给用户]
    
    D -- "未命中 (Miss)" --> G{可随机读 & 小量读取?}
    G -- "是 (Range)" --> H[HTTP Range Request (只读一部分)]
    H --> I[异步触发下一块预读]
    I --> F
    
    G -- "否 (Full Block)" --> J[Singleflight 归并请求]
    J --> K[下载完整 4MB Block]
    K --> L[解压 (Decompress)]
    L --> M[写入本地磁盘缓存]
    M --> F
```

## 关键机制分析 (`pkg/chunk/cached_store.go`)

### 1. 多级缓存分类
- **L1 内存缓存**: 通过 `pagePool` 复用内存页，减少 GC 压力。
- **L2 磁盘缓存**: 
    - 所有的 Block 下载后默认都会持久化到本地磁盘缓存目录。
    - 之后的读取会直接通过 `bcache.load` 访问本地文件。
- **Kernel Page Cache**: JuiceFS 客户端通过普通文件 IO 访问缓存，利用操作系统的页缓存进一步加速。

### 2. 并发控制：Singleflight
当多个线程同时读取同一个未缓存的 Block 时，`store.group.Execute(key, ...)` 会保证只有一个请求真正发往对象存储，其余请求等待结果。这有效防止了“缓存击穿”导致的对象存储压力激增。

### 3. 预读 (Prefetcher)
- 当用户顺序读取文件时，`fetcher` 会根据配置的预读窗口，提前启动后台协程下载后续的 Block。
- 逻辑位于 `cached_store.go` 的 `NewCachedStore` 初始化中。

### 4. 数据压缩处理
- 下载的数据流如果被压缩（LZ4/ZSTD），必须在 `load` 方法中调用 `compressor.Decompress`。
- 如果算法不支持随机访问（如默认的压缩块），即使只读取 1 字节，也必须下载并解压整个 4MB Block。

## Rust 重写思考

1.  **Moka Cache**: 推荐使用 [moka](https://github.com/moka-rs/moka) 作为 L1 内存缓存，它支持高性能的并发访问和多种淘汰策略。
2.  **异步 IO (Tokio)**: 
    - 利用 `tokio::fs` 处理磁盘缓存读取。
    - 利用 `reqwest` 或 `opendal` 处理对象存储的异步 Stream 读取。
3.  **零拷贝 (Zero-copy)**: 尽量使用 `bytes` crate 管理 Buffer，通过引用计数避免在解压和缓存层之间拷贝数据。
4.  **Singleflight 实现**: 
    - 可以参考 `singleflight-async` crate。
    - 本质是维护一个 `HashMap<Key, SharedFuture>`。
5.  **预读策略**: 实现一个基于窗口滑动的预读器，注意控制 `tokio::spawn` 的并发任务数量，防止带宽竞争。
