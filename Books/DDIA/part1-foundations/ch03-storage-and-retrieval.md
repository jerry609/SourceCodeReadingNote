# Chapter 3: Storage and Retrieval

> 存储与检索 - 数据库底层原理

## 核心概念

### 3.1 存储引擎的两大流派

| 类型 | 代表 | 优化目标 |
|:-----|:-----|:---------|
| 日志结构 (Log-Structured) | LSM-Tree, Bitcask | 写入性能 |
| 原地更新 (Update-in-Place) | B-Tree | 读取性能 |

## 日志结构存储

### Hash Index (Bitcask)

```
内存: HashMap { key → (file_id, offset) }
磁盘: 追加写入日志文件
```

**限制**: 所有 key 必须能放入内存

### LSM-Tree (Log-Structured Merge-Tree)

#### 写入流程

```
Write → MemTable (内存) → SSTable (磁盘)
                ↓ (达到阈值)
            Flush to Disk
                ↓
            Compaction (合并)
```

#### SSTable (Sorted String Table)

- Key 有序存储
- 支持范围查询
- 稀疏索引 + 布隆过滤器

#### 读取优化

1. 先查 MemTable
2. 再查 SSTable (从新到旧)
3. 布隆过滤器快速判断 key 不存在

### LSM-Tree 实现

- **LevelDB / RocksDB**: Google 开源
- **Cassandra / HBase**: 列族存储
- **TiKV**: Rust 实现的分布式 KV

## B-Tree

### 结构特点

- 固定大小的页 (通常 4KB)
- 每个节点有 n 个 key 和 n+1 个子指针
- 所有叶子节点在同一层

### 写入流程

```
查找 → 定位叶子页 → 原地更新/分裂
```

### 可靠性保障

- **WAL (Write-Ahead Log)**: 先写日志再修改页
- **Copy-on-Write**: LMDB 采用的方式

## LSM-Tree vs B-Tree

| 特性 | LSM-Tree | B-Tree |
|:-----|:---------|:-------|
| 写入 | 顺序写，更快 | 随机写 |
| 读取 | 可能查多个 SSTable | 一次 B-Tree 查找 |
| 写放大 | Compaction 开销 | 页分裂开销 |
| 空间放大 | 较小（Compaction 回收） | 碎片化 |
| 并发控制 | 简单（不可变文件） | 复杂（页锁） |

## 列式存储

### 适用场景

- OLAP 查询：`SELECT SUM(amount) WHERE date > '2024-01'`
- 只读取需要的列，大幅减少 I/O

### 压缩技术

- **位图编码 (Bitmap Encoding)**
- **游程编码 (Run-Length Encoding)**
- **字典编码 (Dictionary Encoding)**

### 代表系统

- Parquet, ORC (文件格式)
- ClickHouse, Druid (数据库)

## 要点总结

1. 日志结构引擎优化写入，B-Tree 优化读取
2. LSM-Tree 通过 Compaction 回收空间
3. 列式存储适合分析型查询
4. 索引是读写性能的关键权衡

## 思考题

- [ ] 什么场景适合 LSM-Tree？什么场景适合 B-Tree？
- [ ] 为什么 LSM-Tree 需要布隆过滤器？
- [ ] 列式存储为什么适合 OLAP？

## 源码关联

| 概念 | 相关项目 | 实现 |
|:-----|:---------|:-----|
| LSM-Tree | JuiceFS (TiKV 后端) | TiKV 使用 RocksDB |
| 追加写入 | JuiceFS | Chunk 追加写入对象存储 |
| 压缩 | JuiceFS | LZ4/Zstd Block 压缩 |
