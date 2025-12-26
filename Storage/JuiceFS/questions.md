# 疑问与解答 (Q&A)

## 待解决
- [ ] Redis 事务保证：`WATCH` + `MULTI` 如何在分布式环境下保证元数据一致性？
- [ ] TiKV 后端的 MVCC 特性如何被 JuiceFS 利用？
- [ ] Writeback 模式下，后台上传失败后的恢复机制？
- [ ] 如何实现跨客户端的缓存一致性？

## 已解决

### Q1: JuiceFS 的数据如何分层组织？
**A:** JuiceFS 采用四级数据分层：
```
文件 -> Chunk (64MB) -> Block (4MB) -> Page (64KB)
```
- **Chunk**: 逻辑概念，一个文件被切分为多个 Chunk，每个 64MB
- **Block**: 对象存储中的实际存储单元，默认 4MB，是上传/下载的最小粒度
- **Page**: 内存缓冲的最小单元，64KB

*关联代码*: `pkg/chunk/cached_store.go:39-40`
```go
const chunkSize = 1 << 26  // 64M
const pageSize = 1 << 16   // 64K
```

---

### Q2: Redis 后端的数据是如何组织的？
**A:** Redis 使用多种数据结构存储不同类型的元数据：

| 数据类型 | Redis 结构 | Key 格式 |
|:---------|:-----------|:---------|
| 节点属性 | String | `i$inode` |
| 目录项 | Hash | `d$inode` |
| 符号链接 | String | `s$inode` |
| 文件块 | Binary | `c$inode_$indx` |
| 扩展属性 | Hash | `x$inode` |
| BSD 锁 | Hash | `lockf$inode` |

*关联代码*: `pkg/meta/redis.go:57-88`

---

### Q3: Lookup 操作是如何优化的？
**A:** JuiceFS 使用 Lua 脚本将两次 Redis 调用合并为一次原子操作：
1. 普通做法：`HGET d$parent name` -> 获取 inode -> `GET i$inode` -> 获取属性
2. 优化做法：一个 Lua 脚本完成两步操作，减少一次 RTT

*关联代码*: `pkg/meta/lua_scripts.go:20-31`
```lua
local buf = redis.call('HGET', KEYS[1], KEYS[2])
if not buf then error("ENOENT") end
local ino = struct.unpack(">I8", string.sub(buf, 2))
return {ino, redis.call('GET', "i" .. string.format("%.f", ino))}
```

---

### Q4: Singleflight 是什么？为什么需要它？
**A:** Singleflight 是一种请求合并机制，用于防止"缓存击穿"：

**问题场景**：当多个请求同时访问一个未缓存的 Block 时，如果每个请求都去对象存储拉取，会造成：
- 大量重复请求
- 对象存储压力激增
- 带宽浪费

**解决方案**：
```
请求A ──┐
        │
请求B ──┼── Singleflight ──► 只发送一个请求到对象存储 ──► 结果共享给所有等待者
        │
请求C ──┘
```

*关联代码*: `pkg/chunk/singleflight.go:39-65`

---

### Q5: 对象存储中的 Key 是如何命名的？
**A:** JuiceFS 支持两种命名策略：

**普通模式**:
```
chunks/{slice_id/1000000}/{slice_id/1000}/{slice_id}_{block_index}_{block_size}
例如: chunks/0/0/12345_0_4194304
```

**HashPrefix 模式** (更均匀分布):
```
chunks/{slice_id%256 (hex)}/{slice_id/1000000}/{slice_id}_{block_index}_{block_size}
例如: chunks/39/0/12345_0_4194304
```

*关联代码*: `pkg/chunk/cached_store.go:73-78`

---

### Q6: VFS 层如何支持热升级？
**A:** JuiceFS 通过序列化/反序列化文件句柄状态来支持热升级：

1. **升级前**: `dumpAllHandles` 将所有打开的文件句柄状态保存为 JSON
2. **重启后**: `loadAllHandles` 从 JSON 恢复句柄状态，重建 reader/writer

保存的状态包括：
- Inode
- 文件长度
- 打开标志 (flags)
- 锁状态
- 文件偏移

*关联代码*: `pkg/vfs/handle.go:299-426`

---

### Q7: ObjectStorage 接口支持哪些操作？
**A:** `pkg/object/interface.go:77-112` 定义了统一接口：

**基础操作**:
- `Get(key, off, limit)`: 读取对象（支持 Range）
- `Put(key, reader)`: 写入对象
- `Delete(key)`: 删除对象
- `Head(key)`: 获取对象元数据
- `List(prefix, ...)`: 列出对象

**分片上传**:
- `CreateMultipartUpload`: 初始化分片上传
- `UploadPart`: 上传分片
- `UploadPartCopy`: 复制上传
- `CompleteUpload`: 完成上传
- `AbortUpload`: 取消上传

---

### Q8: 支持哪些对象存储后端？
**A:** JuiceFS 支持 50+ 种对象存储 (`pkg/object/` 目录)：

**云厂商**:
- AWS S3, Azure Blob, Google Cloud Storage
- 阿里云 OSS, 腾讯云 COS, 华为云 OBS
- 七牛, 又拍云, 金山云

**分布式存储**:
- MinIO, Ceph (RadosGW), Swift
- HDFS, GlusterFS, NFS

**其他**:
- 本地文件系统, SFTP, WebDAV
- Redis, TiKV, etcd (小文件场景)

---

### Q9: 为什么 Lua 脚本中 inode 有 4503599627370495 的限制？
**A:** 这是 JavaScript/Lua 浮点数精度限制：

- JavaScript 的 `Number` 类型使用 IEEE 754 双精度浮点数
- 双精度浮点数有 52 位有效数字（尾数）
- 最大精确表示的整数是 2^52 - 1 = 4503599627370495

当 inode 超过这个值时，在 Web 界面或某些工具中显示可能不准确，所以 JuiceFS 选择报错而不是返回不精确的结果。

*关联代码*: `pkg/meta/lua_scripts.go:27-29`

---

### Q10: Attr 结构体是如何序列化的？
**A:** 使用大端字节序的二进制格式，总长度 64-80 字节：

```
| Offset | Size | Field      |
|--------|------|------------|
| 0      | 1    | Flags      |
| 1      | 2    | Mode+Type  |  <- Type 在高 4 位
| 3      | 4    | Uid        |
| 7      | 4    | Gid        |
| 11     | 8    | Atime      |
| 19     | 4    | Atimensec  |
| 23     | 8    | Mtime      |
| 31     | 4    | Mtimensec  |
| 35     | 8    | Ctime      |
| 43     | 4    | Ctimensec  |
| 47     | 4    | Nlink      |
| 51     | 8    | Length     |
| 59     | 4    | Rdev       |
| 63     | 8    | Parent     |
| 71     | 4    | AccessACL  |  <- 可选
| 75     | 4    | DefaultACL |  <- 可选
```

*关联代码*: `pkg/meta/interface.go:178-235`
