# JuiceFS Rust Rewrite Architecture Design

## 1. High-Level Architecture

```mermaid
graph TD
    User[User / Application] -->|FUSE / POSIX| FuseLayer[FUSE Layer (fuse-backend-rs)]
    
    subgraph "Core Logic (juicefs-rs)"
        FuseLayer --> VFS[VFS Module (Handle & Inode Mgmt)]
        
        VFS -->|Metadata Ops| Meta[Meta Engine Trait]
        VFS -->|Data Ops| Chunk[Chunk Store Trait]
        
        subgraph "Metadata Engines"
            Meta -->|Redis Protocol| Redis[Redis Backend]
            Meta -->|SQLx| SQL[SQL Backend]
        end
        
        subgraph "Data Storage"
            Chunk -->|L1/L2 Cache| Cache[Cache Manager (Moka/Disk)]
            Cache -->|Prefetch/Writeback| Buffer[Buffer Pool]
            Buffer -->|OpenDAL Operator| Object[Object Storage (S3/OSS/etc)]
        end
    end
    
    Redis -->|TCP| RedisServer[Redis Server]
    Object -->|HTTP| Cloud[Cloud Storage]
```

## 2. Crate Structure (Workspace)

建议使用 Cargo Workspace 组织项目：

| Crate Name | 职责描述 | 对应 Go 模块 |
| :--- | :--- | :--- |
| **`juicefs-core`** | 核心库，定义所有 Trait (`Meta`, `ChunkStore`) 和核心逻辑。 | `pkg/juicefs` |
| **`juicefs-meta`** | 元数据引擎实现 (Redis, SQL)。 | `pkg/meta` |
| **`juicefs-chunk`** | 数据分块、缓存、对象存储交互逻辑。 | `pkg/chunk`, `pkg/object` |
| **`juicefs-fuse`** | FUSE 挂载入口，实现 `fuse-backend-rs` 接口。 | `pkg/vfs`, `pkg/fs` |
| **`juicefs-cli`** | 命令行工具 (`mount`, `format`, `bench`)。 | `cmd/` |

## 3. Core Traits Definition

### 3.1 Metadata Engine (`juicefs-meta`)

```rust
use async_trait::async_trait;

#[async_trait]
pub trait MetaEngine: Send + Sync {
    // Inode Operations
    async fn lookup(&self, parent: Ino, name: &str) -> Result<Entry>;
    async fn getattr(&self, ino: Ino) -> Result<Attr>;
    async fn setattr(&self, ino: Ino, attr: SetAttr) -> Result<Attr>;
    
    // Directory Operations
    async fn mknod(&self, parent: Ino, name: &str, mode: u32) -> Result<Entry>;
    async fn unlink(&self, parent: Ino, name: &str) -> Result<()>;
    async fn readdir(&self, ino: Ino, offset: i64) -> Result<Vec<Entry>>;
    
    // Chunk Mapping
    async fn new_slice(&self) -> Result<u64>;
    async fn write_slice(&self, ino: Ino, chunk_idx: u32, slice: SliceInfo) -> Result<()>;
    async fn read_slice(&self, ino: Ino, chunk_idx: u32) -> Result<Vec<SliceInfo>>;
}
```

### 3.2 Chunk Store (`juicefs-chunk`)

```rust
use bytes::Bytes;

#[async_trait]
pub trait ChunkStore: Send + Sync {
    // Main IO
    async fn read_at(&self, chunk_id: u64, offset: u64, len: u32) -> Result<Bytes>;
    async fn write_at(&self, chunk_id: u64, offset: u64, data: Bytes) -> Result<u32>;
    
    // Lifecycle
    async fn flush(&self, chunk_id: u64) -> Result<()>;
    async fn remove(&self, chunk_id: u64) -> Result<()>;
}
```

## 4. Technical Stack Selection

| Component | Go Implementation | Rust Selection | Rationale |
| :--- | :--- | :--- | :--- |
| **FUSE** | `hanwen/go-fuse` | **`fuse-backend-rs`** | 阿里开源，生产级，支持 virtio-fs，架构优于 `fuser`。 |
| **Object Storage** | `pkg/object` (Custom wrappers) | **`opendal`** | Apache 顶级项目，统一抽象，无需维护几十个 SDK。 |
| **Metadata DB** | `go-redis`, `sql` | **`redis-rs`**, **`sqlx`** | Rust 生态标准选择，支持 Async。 |
| **Async Runtime** | Goroutines | **`tokio`** | 事实标准。 |
| **In-Memory Cache** | Custom implementation | **`moka`** | 高性能并发缓存，支持 W-TinyLFU。 |
| **Binary Serialize** | Custom `Marshal` | **`bincode`** | 极快，二进制兼容性好，适合模拟 JuiceFS 的紧凑格式。 |
| **Error Handling** | `error`, `syscall.Errno` | **`thiserror`**, **`anyhow`** | 更好的错误上下文管理。 |

## 5. Implementation Roadmap

1.  **Phase 1: The Foundation**
    *   Setup workspace.
    *   Define `MetaEngine` trait.
    *   Implement `RedisMeta` (basic Lookup/GetAttr).
    *   Unit tests with `testcontainers` (Redis).

2.  **Phase 2: The Data Layer**
    *   Setup `opendal` with `fs` backend (local disk) for testing.
    *   Implement `ChunkStore` logic (Split chunk -> slice).
    *   Implement Write path (Memory buffer -> Flush).

3.  **Phase 3: FUSE Integration**
    *   Implement `fuse_backend_rs::Filesystem` trait.
    *   Connect `VFS` to `Meta` and `Chunk`.
    *   Mount and `ls`, `touch`, `echo "hello" > file`.

4.  **Phase 4: Optimization**
    *   Add `moka` cache.
    *   Implement Prefetching.
    *   Implement background Writeback.
