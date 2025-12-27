# DDIA 概念与源码交叉引用

> 将 DDIA 的理论概念与实际项目源码对照，加深理解。

## Part I: Foundations - 数据系统基础

### Chapter 3: Storage and Retrieval

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| LSM-Tree | TiKV 后端使用 RocksDB | `pkg/meta/tkv_tikv.go` |
| 追加写入 | Chunk 只追加不修改 | `pkg/chunk/cached_store.go` |
| 压缩算法 | LZ4/Zstd Block 压缩 | `pkg/compress/` |
| 布隆过滤器 | 对象存储 Key 查找优化 | N/A (对象存储内置) |

### Chapter 4: Encoding and Evolution

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 二进制编码 | Attr 结构体序列化 | `pkg/meta/interface.go:178-235` |
| Schema 演进 | 元数据格式版本兼容 | `pkg/meta/format.go` |
| 大端序 | 属性字段编码 | `pkg/utils/buffer.go` |

## Part II: Distributed Data - 分布式数据

### Chapter 5: Replication

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 主从复制 | Redis Sentinel 模式 | `pkg/meta/redis.go` |
| 多副本 | 对象存储内置 (S3 11个9) | `pkg/object/` |
| 异步复制 | Writeback 模式 | `pkg/chunk/cached_store.go:420` |
| Quorum | TiKV Raft 多数派 | `pkg/meta/tkv_tikv.go` |

### Chapter 6: Partitioning

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 按 Key 分区 | Chunk 64MB 分片 | `pkg/chunk/cached_store.go:39` |
| Hash 分布 | HashPrefix 对象 Key | `pkg/chunk/cached_store.go:73-78` |
| 热点处理 | Singleflight 合并请求 | `pkg/chunk/singleflight.go` |
| 元数据路由 | TiKV PD (Placement Driver) | `pkg/meta/tkv_tikv.go` |

### Chapter 7: Transactions

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 乐观并发控制 | Redis WATCH + MULTI | `pkg/meta/redis.go:1116-1165` |
| 悲观锁 | SQL SELECT FOR UPDATE | `pkg/meta/sql.go:1038-1090` |
| MVCC | TiKV Percolator 模型 | `pkg/meta/tkv_tikv.go:246-275` |
| 2PC | TiKV 两阶段提交 | `pkg/meta/tkv_tikv.go` |

### Chapter 8: The Trouble with Distributed Systems

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 超时重试 | 对象存储请求重试 | `pkg/chunk/cached_store.go:376-394` |
| 指数退避 | 事务重试策略 | `pkg/meta/redis.go:1128` |
| 心跳检测 | Session 心跳机制 | `pkg/meta/base.go` |
| 租约 | Session 过期处理 | `pkg/meta/redis.go` |

### Chapter 9: Consistency and Consensus

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| Raft 共识 | TiKV 多数派复制 | `pkg/meta/tkv_tikv.go` |
| 分布式锁 | Flock/Plock 实现 | `pkg/meta/redis_lock.go` |
| Leader 选举 | Redis Sentinel | 外部依赖 |
| 线性一致性读 | TiKV Read Index | `pkg/meta/tkv_tikv.go` |

## Part III: Derived Data - 派生数据

### Chapter 10: Batch Processing

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 批量处理 | `juicefs gc --compact` | `cmd/gc.go` |
| 并行扫描 | 多线程元数据遍历 | `pkg/meta/base.go` |
| ETL | `juicefs sync` 数据同步 | `pkg/sync/sync.go` |
| 分片上传 | Multipart Upload | `pkg/sync/sync.go:661-671` |

### Chapter 11: Stream Processing

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 变更通知 | 文件系统事件监听 | `pkg/vfs/` |
| 日志 | Access Log | `pkg/vfs/accesslog.go` |
| 异步处理 | Writeback 延迟上传 | `pkg/chunk/cached_store.go:1084-1105` |
| 消息队列 | 待上传任务队列 | `pkg/chunk/cached_store.go:pendingCh` |

### Chapter 12: The Future of Data Systems

| DDIA 概念 | JuiceFS 实现 | 源码位置 |
|:----------|:-------------|:---------|
| 派生数据 | 缓存 = 派生自对象存储 | `pkg/chunk/disk_cache.go` |
| 幂等操作 | Slice ID 唯一性 | `pkg/meta/base.go` |
| 数据删除 | `juicefs gc` 垃圾回收 | `cmd/gc.go` |
| 端到端校验 | CRC32 数据校验 | `pkg/chunk/disk_cache.go` |

## 核心模式对照表

| 模式 | DDIA 章节 | JuiceFS 实现 |
|:-----|:----------|:-------------|
| 读放大优化 | Ch3 LSM-Tree | 缓存层减少对象存储访问 |
| 写放大优化 | Ch3 LSM-Tree | Writeback 合并写入 |
| 请求合并 | Ch6 热点处理 | Singleflight |
| 一致性 vs 可用性 | Ch9 CAP | 元数据强一致，数据最终一致 |
| 分层存储 | Ch3 存储层次 | 内存 → 磁盘缓存 → 对象存储 |

## 延伸阅读

### 相关论文

| 论文 | 相关章节 | 说明 |
|:-----|:---------|:-----|
| Google File System | Ch3, Ch5 | 分布式文件系统基础 |
| Bigtable | Ch3 | LSM-Tree 在分布式存储的应用 |
| Dynamo | Ch5, Ch6 | 无主复制、一致性哈希 |
| Raft | Ch9 | 共识算法 |
| Percolator | Ch7 | 分布式事务 |

### 相关项目源码

| 项目 | 相关概念 | 推荐阅读 |
|:-----|:---------|:---------|
| Redis | 复制、事务、分布式锁 | 哨兵、Cluster 模块 |
| TiKV | Raft、MVCC、分布式事务 | raftstore 模块 |
| RocksDB | LSM-Tree、Compaction | db_impl.cc |
| etcd | Raft、租约、Watch | raft 模块 |
