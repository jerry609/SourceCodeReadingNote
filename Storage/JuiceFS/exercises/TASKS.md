# 🦁 模拟实战：JuiceFS Rust 重写任务清单

> **角色扮演说明**: 
> 你现在是 **JuiceFS Rust Rewrite** 项目的技术负责人 (Tech Lead)。
> 下面是你为团队成员（其实就是你自己）拆解的开发任务。
> 请按照顺序领取任务，每完成一个任务，不仅要写代码，还要通过相应的单元测试。

---

## Milestone 1: 元数据引擎的核心 (The Brain)
**目标**: 实现一个基于 Redis 的元数据引擎，支持基本的文件增删改查。
**预估周期**: 2 Weeks

### 🟢 Task 1.1: 定义 `MetaEngine` Trait
*   **优先级**: P0
*   **背景**: 我们需要支持多种元数据后端（Redis, SQL, TiKV），所以必须先定义一个通用的接口。
*   **需求**:
    1. 在 `juicefs-core` crate 中定义 `MetaEngine` trait。
    2. 包含基础方法：`lookup`, `getattr`, `mknod`, `mkdir`。
    3. 定义核心数据结构：`Ino` (u64), `Attr` (struct), `Entry` (struct)。
    4. 所有的 IO 方法必须是 `async` 的。
*   **参考**: `pkg/meta/interface.go`
*   **验收标准**:
    - [ ] Trait 定义可以通过 `cargo check`。
    - [ ] `Attr` 结构体实现了 `bincode` 序列化/反序列化。

### 🟡 Task 1.2: Redis Schema 设计与 Lua 脚本封装
*   **优先级**: P0
*   **背景**: 为了性能，我们需要用 Lua 脚本合并 Redis 请求。
*   **需求**:
    1. 实现 `RedisMeta` 结构体。
    2. 编写 `lookup.lua`：输入 parentIno, name，输出 inode, attr。
    3. 使用 `redis-rs` 的 `Script` 模块在 `new()` 时预加载脚本。
*   **参考**: `pkg/meta/redis.go` 中的 `scriptLookup` 变量。
*   **验收标准**:
    - [ ] 单元测试：启动一个 Redis 容器，加载脚本，调用成功返回结果。

### 🔴 Task 1.3: 实现 Lookup 与 Mknod
*   **优先级**: P1
*   **背景**: 这是文件系统最基础的两个操作：创建文件和查找文件。
*   **需求**:
    1. 实现 `mknod`:
        - 生成新 inode ID (原子递增 `nextInode` key)。
        - 写入 `i<inode>` (属性)。
        - 写入 `d<parent>` (目录项)。
    2. 实现 `lookup`:
        - 调用 Task 1.2 中的 Lua 脚本。
*   **提示**: 注意 `Attr` 中的 `Mode`, `Uid`, `Gid`, `Time` 字段的处理。
*   **验收标准**:
    - [ ] 集成测试：`mknod(root, "file.txt")` 成功 -> `lookup(root, "file.txt")` 能查到刚创建的 inode。

---

## Milestone 2: 数据存储层 (The Body)
**目标**: 对接对象存储，实现数据的切片与写入。
**预估周期**: 2 Weeks

### 🟢 Task 2.1: 集成 OpenDAL
*   **优先级**: P0
*   **背景**: 我们不重复造轮子，直接用 OpenDAL 对接 S3/fs。
*   **需求**:
    1. 在 `juicefs-chunk` crate 中引入 `opendal`。
    2. 实现一个工厂方法，根据配置返回 `opendal::Operator`。
*   **参考**: `pkg/object` (Go 版本不需要看太多，看 Rust 文档即可)。
*   **验收标准**:
    - [ ] 单元测试：配置为 `fs` (本地文件) 后端，能成功 `write` 和 `read` 一个文件。

### 🟡 Task 2.2: 实现 Slice 逻辑 (Write Path)
*   **优先级**: P1
*   **背景**: JuiceFS 将大文件切分为 Chunk (64MB) -> Block (4MB)。你需要实现这个切分逻辑。
*   **需求**:
    1. 定义 `ChunkStore` trait。
    2. 实现 `write_at(chunk_id, offset, data)`。
    3. 逻辑：
        - 计算数据落在哪个 4MB Block。
        - 暂时在内存中 Buffer (使用 `Vec<u8>` 或 `BytesMut`)。
        - 当 Buffer 满 4MB 时，调用 OpenDAL 写入对象存储。
*   **参考**: `pkg/chunk/cached_store.go` 中的 `wSlice` 结构体。
*   **验收标准**:
    - [ ] 单元测试：写入 10MB 数据，验证后端对象存储中是否生成了 3 个对象 (4MB + 4MB + 2MB)。

---

## Milestone 3: FUSE 集成 (The Face)
**目标**: 让用户能通过 `ls`, `cat` 等命令操作我们的文件系统。
**预估周期**: 1 Week

### 🔴 Task 3.1: 实现 `fuse-backend-rs` 接口
*   **优先级**: P1
*   **背景**: 这是连接内核与我们代码的桥梁。
*   **需求**:
    1. 创建 `juicefs-fuse` crate。
    2. 实现 `fuse_backend_rs::api::filesystem::FileSystem` trait。
    3. 将 `lookup`, `getattr` 转发给 `MetaEngine`。
    4. 将 `read`, `write` 转发给 `ChunkStore`。
*   **参考**: `pkg/vfs/vfs.go`。
*   **验收标准**:
    - [ ] 能够成功 Mount。
    - [ ] `touch test.txt` 成功。
    - [ ] `ls -l` 能看到正确的文件属性。

---

## 进阶挑战 (Bonus)

*   **Task 4.1**: 实现客户端内存缓存 (`moka` 集成)。
*   **Task 4.2**: 实现数据压缩 (LZ4)。
*   **Task 4.3**: 实现 `rename` 操作（注意：Redis 中 Rename 需要处理原子性，可能需要复杂的 Lua 脚本）。
