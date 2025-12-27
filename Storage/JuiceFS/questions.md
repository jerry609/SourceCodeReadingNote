# 疑问与解答 (Q&A)

## 待解决
- [ ] Redis 事务保证：`WATCH` + `MULTI` 如何在分布式环境下保证元数据一致性？
- [ ] TiKV 后端的 MVCC 特性如何被 JuiceFS 利用？
- [ ] 如何实现跨客户端的缓存一致性（非 Redis Client Tracking 场景）？

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

---

### Q11: Writeback 模式下上传失败如何恢复？
**A:** JuiceFS 通过 staging 目录实现故障恢复：

1. **写入时**: 数据先写入本地 staging 目录，立即返回成功
2. **后台上传**: 异步上传到对象存储
3. **失败处理**: 上传失败的文件加入 `delayedStaging` 队列，定期重试
4. **重启恢复**: 客户端启动时扫描 staging 目录，重新上传未完成的文件

```go
// pkg/chunk/cached_store.go:420-458
if s.writeback && blen < s.store.conf.WritebackThresholdSize {
    stagingPath, err := s.store.bcache.stage(key, block.Data)
    if err == nil {
        s.errors <- nil  // 立即返回成功
        s.store.addDelayedStaging(key, stagingPath, time.Now(), false)
        return
    }
}
```

---

### Q12: JuiceFS 支持哪些缓存淘汰策略？
**A:** 三种策略，通过 `--cache-eviction` 配置：

| 策略 | 数据结构 | 淘汰复杂度 | 适用场景 |
|:-----|:---------|:-----------|:---------|
| `none` | HashMap | N/A | 缓存永不过期 |
| `2-random` | HashMap | O(1) | 内存敏感，随机采样 |
| `lru` | HashMap + MinHeap | O(log n) | 热点数据明显 |

**2-random 算法**: 每遍历 2 个缓存项，选择较旧的一个淘汰（类似 Redis 的近似 LRU）。

*关联代码*: `pkg/chunk/cache_eviction.go:54-71`

---

### Q13: 预读 (Prefetch) 是如何实现的？
**A:** 使用 Worker Pool + Channel + Map 实现：

```go
// pkg/chunk/prefetch.go:23-63
type prefetcher struct {
    pending chan string       // 任务队列（buffer=10）
    busy    map[string]bool   // 去重：正在处理的 key
    op      func(key string)  // 实际预读函数
}
```

**关键设计**:
- **并发限制**: 固定数量的 worker goroutine
- **去重**: `busy` map 防止重复预读同一块
- **非阻塞**: channel 满时直接丢弃任务

**触发时机**: 成功读取一个块后，触发下一个块的预读 (`cached_store.go:743`)

---

### Q14: FUSE writeback_cache 模式下为什么 O_WRONLY 也需要 reader？
**A:** 这是 FUSE 内核缓存的特性决定的：

在 `writeback_cache` 模式下，内核会缓存写入数据到 page cache。当需要对齐页面时（如部分页写入），内核可能会对只写打开的文件发起读请求。

```go
// pkg/vfs/handle.go:245-249
case syscall.O_WRONLY: // FUSE writeback_cache mode need reader even for WRONLY
    fallthrough
case syscall.O_RDWR:
    h.reader = v.reader.Open(inode, length)
    h.writer = v.writer.Open(inode, length)
```

如果不创建 reader，内核的读请求会失败，导致写入失败。

---

### Q15: JuiceFS 如何通知内核使缓存失效？
**A:** 通过 FUSE 的 `EntryNotify` 接口（仅 Linux）：

```go
// pkg/fuse/fuse.go:533-535
v.InvalidateEntry = func(parent Ino, name string) syscall.Errno {
    return syscall.Errno(fssrv.EntryNotify(uint64(parent), name))
}
```

**三类失效**:
| 类型 | 触发时机 | 作用 |
|:-----|:---------|:-----|
| Entry | 删除/重命名后 | 使目录项缓存失效 |
| Attr | 写入/truncate 后 | 使属性缓存失效 |
| Chunk | 写入/compact 后 | 使数据缓存失效 |

---

### Q16: Chunk Compact 是如何触发的？
**A:** 两个触发点：

**读取时** (`base.go:1928`):
```go
if len(ss) >= 5 || len(*slices) >= 5 {
    go m.compactChunk(inode, indx, false, false)  // 异步
}
```

**写入时** (`base.go:1982`):
```go
if numSlices % 100 == 99 || numSlices > 350 {
    if numSlices < maxSlices {
        go m.compactChunk(...)  // 异步
    } else {
        m.compactChunk(...)     // 同步阻塞
    }
}
```

**去重**: 使用 `compacting` map 防止同一 chunk 被重复压缩。

---

### Q17: Compact 时如何决定跳过哪些 slice？
**A:** `skipSome` 函数实现智能跳过：

```go
// pkg/meta/slice.go:183-210
func skipSome(chunk []*slice) int {
    for skipped < len(chunk) {
        first := chunk[skipped]
        // 跳过条件：slice > 1MB 且占总大小 > 20%
        if first.len < (1<<20) || first.len*5 < size {
            break  // 太小，需要压缩
        }
        skipped++
    }
    return skipped
}
```

**原理**: 大 slice 已经是完整的数据块，压缩收益小；小 slice 需要合并以减少碎片。

---

### Q18: 分片上传 (Multipart Upload) 的分片大小如何选择？
**A:** 动态计算：

```go
// pkg/sync/sync.go:661-671
func choosePartSize(upload *MultipartUpload, size int64) int64 {
    partSize := int64(upload.MinPartSize)  // 默认 5MB
    // 如果分片太小导致数量超限，自动调大
    if size > partSize * int64(upload.MaxCount) {
        partSize = size / int64(upload.MaxCount)
        partSize = ((partSize-1)>>20 + 1) << 20  // 向上对齐到 MB
    }
    return partSize
}
```

| 文件大小 | 分片大小 | 分片数量 |
|:---------|:---------|:---------|
| < 50GB | 5MB | < 10000 |
| 50GB - 500GB | 5MB - 50MB | ~10000 |
| > 500GB | 动态调整 | 10000 |

---

### Q19: Slice 重叠是如何处理的？
**A:** 使用二叉树 + `cut` 算法：

```go
// pkg/meta/slice.go:55-79
func (s *slice) cut(pos uint32) (left, right *slice) {
    if pos <= s.pos {
        // 在左边切
        left, s.left = s.left.cut(pos)
        return left, s
    } else if pos < s.pos + s.len {
        // 在中间切
        right = newSlice(pos, s.id, s.size, s.off+l, s.len-l)
        s.len = l
        return s, right
    } else {
        // 在右边切
        s.right, right = s.right.cut(pos)
        return s, right
    }
}
```

每个新写入的 slice 会 "切开" 已有的 slice，确保没有重叠。最终通过中序遍历得到有序的 slice 列表。

---

### Q20: Page Pool 是如何实现内存复用的？
**A:** 使用带缓冲的 channel 作为对象池：

```go
// pkg/chunk/page.go:38-55
var pagePool = make(chan *Page, 128)

func allocPage() *Page {
    select {
    case p := <-pagePool:
        return p
    default:
        return &Page{Data: make([]byte, pageSize)}
    }
}

func freePage(p *Page) {
    select {
    case pagePool <- p:
    default:
        // 池满，让 GC 回收
    }
}
```

**优势**: 避免频繁 GC，减少内存分配开销。
**池大小**: 128 个 Page × 64KB = 8MB 内存。
