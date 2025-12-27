# pkg/chunk 模块分析

## 职责
`pkg/chunk` 是 JuiceFS 的数据层核心，负责数据的切片、缓存、压缩和与对象存储的交互。它屏蔽了对象存储的复杂性，为上层 VFS 提供统一的读写接口。

**源码位置**: `pkg/chunk/` 目录

## 核心文件结构

| 文件 | 职责 |
|:-----|:-----|
| `chunk.go` | 核心接口定义 (ChunkStore, Reader, Writer) |
| `cached_store.go` | 缓存存储实现 (核心!) |
| `page.go` | 内存页管理 |
| `disk_cache.go` | 磁盘缓存实现 |
| `mem_cache.go` | 内存缓存实现 |
| `singleflight.go` | 请求合并 (防止缓存击穿) |
| `prefetch.go` | 预读逻辑 |
| `cache_eviction.go` | 缓存淘汰策略 |

## 核心常量

```go
// pkg/chunk/cached_store.go:39-41
const chunkSize = 1 << 26  // 64M - 一个 Chunk 的大小
const pageSize = 1 << 16   // 64K - 一个 Page 的大小
const SlowRequest = time.Second * time.Duration(10)
```

**数据分层结构**:
```
文件 -> Chunk (64MB) -> Block (4MB 默认) -> Page (64KB)
```

## 核心接口

### ChunkStore 接口
```go
// pkg/chunk/chunk.go:38-47
type ChunkStore interface {
    NewReader(id uint64, length int) Reader
    NewWriter(id uint64) Writer
    Remove(id uint64, length int) error
    FillCache(id uint64, length uint32) error
    EvictCache(id uint64, length uint32) error
    CheckCache(id uint64, length uint32, handler func(exists bool, loc string, size int)) error
    UsedMemory() int64
    UpdateLimit(upload, download int64)
}
```

### Reader 接口
```go
// pkg/chunk/chunk.go:24-26
type Reader interface {
    ReadAt(ctx context.Context, p *Page, off int) (int, error)
}
```

### Writer 接口
```go
// pkg/chunk/chunk.go:28-36
type Writer interface {
    io.WriterAt
    ID() uint64
    SetID(id uint64)
    SetWriteback(enabled bool)
    FlushTo(offset int) error
    Finish(length int) error
    Abort()
}
```

## 核心实现：cachedStore

### 结构体定义
```go
// pkg/chunk/cached_store.go (推断自代码分析)
type cachedStore struct {
    storage    object.ObjectStorage  // 对象存储后端
    conf       Config
    bcache     *diskCache            // 磁盘缓存
    mcache     *memCache             // 内存缓存
    compressor compress.Compressor   // 压缩器
    group      *Controller           // Singleflight 控制器

    // 限速器
    upLimit    *ratelimit.Bucket
    downLimit  *ratelimit.Bucket

    // 后台上传
    currentUpload chan struct{}      // 并发控制
    pendingCh     chan *pendingItem  // 待上传队列

    // 指标
    cacheHits      prometheus.Counter
    cacheMiss      prometheus.Counter
    objectReqErrors prometheus.Counter
    // ...
}
```

## 读取路径 (rSlice)

### rSlice 结构体
```go
// pkg/chunk/cached_store.go:54-59
type rSlice struct {
    id     uint64
    length int
    store  *cachedStore
}
```

### 对象存储 Key 生成
```go
// pkg/chunk/cached_store.go:73-78
func (s *rSlice) key(indx int) string {
    if s.store.conf.HashPrefix {
        // 使用 hash 前缀分散存储
        return fmt.Sprintf("chunks/%02X/%v/%v_%v_%v", s.id%256, s.id/1000/1000, s.id, indx, s.blockSize(indx))
    }
    // 按 ID 分目录
    return fmt.Sprintf("chunks/%v/%v/%v_%v_%v", s.id/1000/1000, s.id/1000, s.id, indx, s.blockSize(indx))
}
```

### ReadAt 核心逻辑
```go
// pkg/chunk/cached_store.go:96-178 (简化版)
func (s *rSlice) ReadAt(ctx context.Context, page *Page, off int) (n int, err error) {
    indx := s.index(off)
    boff := off % s.store.conf.BlockSize
    key := s.key(indx)

    // 1. 尝试从磁盘缓存读取
    if s.store.conf.CacheEnabled() {
        r, err := s.store.bcache.load(key)
        if err == nil {
            n, err = r.ReadAt(p, int64(boff))
            r.Close()
            if err == nil {
                s.store.cacheHits.Add(1)
                return n, nil
            }
        }
    }

    s.store.cacheMiss.Add(1)

    // 2. 小量随机读 -> Range Request
    if s.store.seekable && (boff > 0 && len(p) <= blockSize/4) {
        return s.store.loadRange(ctx, key, page, boff)
    }

    // 3. 完整块读取 -> Singleflight
    block, err := s.store.group.Execute(key, func() (*Page, error) {
        tmp := NewOffPage(blockSize)
        err = s.store.load(ctx, key, tmp, shouldCache, false)
        return tmp, err
    })
    defer block.Release()

    copy(p, block.Data[boff:])
    return len(p), nil
}
```

## 写入路径 (wSlice)

### wSlice 结构体
```go
// pkg/chunk/cached_store.go:237-245
type wSlice struct {
    rSlice
    pages       [][]*Page   // 二维数组: [Block Index][Page 列表]
    uploaded    int         // 已上传偏移
    errors      chan error  // 上传错误通道
    uploadError error
    pendings    int         // 待上传数量
    writeback   bool        // 是否启用回写
}
```

### WriteAt 核心逻辑
```go
// pkg/chunk/cached_store.go:264-306
func (s *wSlice) WriteAt(p []byte, off int64) (n int, err error) {
    // 边界检查
    if int(off)+len(p) > chunkSize {
        return 0, fmt.Errorf("write out of chunk boundary")
    }

    // 填充空洞
    if s.length < int(off) {
        zeros := make([]byte, int(off)-s.length)
        s.WriteAt(zeros, int64(s.length))
    }

    // 写入到 Page
    for n < len(p) {
        indx := s.index(int(off) + n)
        boff := (int(off) + n) % s.store.conf.BlockSize
        bi := boff / pageSize
        bo := boff % pageSize

        // 分配或获取 Page
        var page *Page
        if bi < len(s.pages[indx]) {
            page = s.pages[indx][bi]
        } else {
            page = allocPage(pageSize)
            s.pages[indx] = append(s.pages[indx], page)
        }

        n += copy(page.Data[bo:], p[n:])
    }

    if int(off)+n > s.length {
        s.length = int(off) + n
    }
    return n, nil
}
```

### 上传逻辑
```go
// pkg/chunk/cached_store.go:397-464
func (s *wSlice) upload(indx int) {
    key := s.key(indx)
    pages := s.pages[indx]
    s.pages[indx] = nil
    s.pendings++

    go func() {
        // 组装 Block
        var block *Page
        if len(pages) == 1 {
            block = pages[0]
        } else {
            block = NewOffPage(blen)
            for _, b := range pages {
                copy(block.Data[off:], b.Data)
                freePage(b)
            }
        }

        // Writeback 模式: 先写磁盘缓存
        if s.writeback && blen < s.store.conf.WritebackThresholdSize {
            stagingPath, err := s.store.bcache.stage(key, block.Data)
            if err == nil {
                s.errors <- nil
                // 后台异步上传
                s.store.addDelayedStaging(key, stagingPath, time.Now(), false)
                return
            }
        }

        // 直接上传
        s.store.currentUpload <- struct{}{}
        defer func() { <-s.store.currentUpload }()
        s.errors <- s.store.upload(key, block, s)
    }()
}
```

## Singleflight 实现

防止缓存击穿，多个请求同一 Block 时只发起一次实际请求：

```go
// pkg/chunk/singleflight.go:21-65
type request struct {
    wg   sync.WaitGroup
    val  *Page
    dups int
    err  error
}

type Controller struct {
    sync.Mutex
    rs map[string]*request
}

func (con *Controller) Execute(key string, fn func() (*Page, error)) (*Page, error) {
    con.Lock()
    // 如果已有请求在处理，等待结果
    if c, ok := con.rs[key]; ok {
        c.dups++
        con.Unlock()
        c.wg.Wait()
        return c.val, c.err
    }

    // 创建新请求
    c := new(request)
    c.wg.Add(1)
    con.rs[key] = c
    con.Unlock()

    c.val, c.err = fn()

    con.Lock()
    // 为等待者增加引用计数
    for i := 0; i < c.dups; i++ {
        c.val.Acquire()
    }
    delete(con.rs, key)
    con.Unlock()

    c.wg.Done()
    return c.val, c.err
}
```

## Page 内存管理

### Page 结构体
```go
// pkg/chunk/page.go (推断)
type Page struct {
    Data []byte
    refs int32  // 引用计数
}

func (p *Page) Acquire() { atomic.AddInt32(&p.refs, 1) }
func (p *Page) Release() {
    if atomic.AddInt32(&p.refs, -1) == 0 {
        // 归还到 pool 或释放
    }
}
```

### Page Pool
```go
// pkg/chunk/cached_store.go:210-234
var pagePool = make(chan *Page, 128)

func allocPage(sz int) *Page {
    if sz != pageSize {
        return NewOffPage(sz)
    }
    select {
    case p := <-pagePool:
        return p
    default:
        return NewOffPage(pageSize)
    }
}

func freePage(p *Page) {
    if cap(p.Data) != pageSize {
        p.Release()
        return
    }
    select {
    case pagePool <- p:
    default:
        p.Release()
    }
}
```

## 缓存架构

### 多级缓存

```
┌─────────────────┐
│   Memory Cache  │  <- L1: 热点数据 (moka/内置)
│   (mem_cache)   │
└────────┬────────┘
         │
┌────────▼────────┐
│   Disk Cache    │  <- L2: 持久化缓存
│  (disk_cache)   │
└────────┬────────┘
         │
┌────────▼────────┐
│   Page Cache    │  <- L3: 操作系统页缓存
│    (kernel)     │
└────────┬────────┘
         │
┌────────▼────────┐
│ Object Storage  │  <- 持久化存储
│   (S3/OSS...)   │
└─────────────────┘
```

### 缓存配置
```go
type Config struct {
    BlockSize           int           // 默认 4MB
    CacheDir            string        // 缓存目录
    CacheSize           int64         // 缓存大小限制
    FreeSpace           float32       // 保留空闲空间比例
    CacheMode           CacheMode     // 缓存模式
    AutoCreate          bool
    Compress            string        // 压缩算法
    MaxUpload           int           // 最大并发上传
    MaxRetries          int           // 重试次数
    UploadDelay         time.Duration // 延迟上传
    Writeback           bool          // 回写模式
    WritebackThreshold  int           // 回写阈值
    Prefetch            int           // 预读块数
    PutTimeout          time.Duration
    GetTimeout          time.Duration
    // ...
}
```

## 数据压缩

JuiceFS 支持多种压缩算法 (`pkg/compress/`)：

| 算法 | 压缩比 | 速度 | 适用场景 |
|:-----|:-------|:-----|:---------|
| LZ4 | 中 | 最快 | 默认推荐 |
| ZSTD | 高 | 较快 | 空间敏感 |
| Gzip | 高 | 慢 | 兼容性 |
| None | 无 | - | 已压缩数据 |

## Rust 重写思考

### 库选型
- **对象存储**: `opendal` (见 pkg_object.md)
- **内存缓存**: `moka` (高性能并发 LRU)
- **磁盘缓存**: 自实现或 `cacache`
- **压缩**: `lz4`, `zstd` crates
- **异步**: `tokio`

### ChunkStore Trait 设计
```rust
#[async_trait]
pub trait ChunkStore: Send + Sync {
    fn new_reader(&self, id: u64, length: usize) -> Box<dyn ChunkReader>;
    fn new_writer(&self, id: u64) -> Box<dyn ChunkWriter>;
    async fn remove(&self, id: u64, length: usize) -> Result<()>;
    async fn fill_cache(&self, id: u64, length: u32) -> Result<()>;
    async fn evict_cache(&self, id: u64, length: u32) -> Result<()>;
    fn used_memory(&self) -> i64;
}

#[async_trait]
pub trait ChunkReader: Send + Sync {
    async fn read_at(&self, buf: &mut [u8], off: usize) -> Result<usize>;
}

pub trait ChunkWriter: Send + Sync {
    fn write_at(&mut self, buf: &[u8], off: i64) -> Result<usize>;
    async fn flush_to(&mut self, offset: usize) -> Result<()>;
    async fn finish(&mut self, length: usize) -> Result<()>;
    fn abort(&mut self);
}
```

### Page 管理
```rust
use std::sync::Arc;
use bytes::BytesMut;

pub struct Page {
    data: BytesMut,
}

impl Page {
    pub fn new(size: usize) -> Self {
        Self { data: BytesMut::zeroed(size) }
    }

    pub fn slice(&self, start: usize, len: usize) -> &[u8] {
        &self.data[start..start+len]
    }
}

// 使用 Arc 实现共享
type SharedPage = Arc<Page>;
```

### Singleflight 实现
```rust
use tokio::sync::broadcast;
use dashmap::DashMap;

pub struct Singleflight<T> {
    requests: DashMap<String, broadcast::Sender<T>>,
}

impl<T: Clone> Singleflight<T> {
    pub async fn execute<F, Fut>(&self, key: &str, f: F) -> T
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = T>,
    {
        // 检查是否已有进行中的请求
        if let Some(tx) = self.requests.get(key) {
            let mut rx = tx.subscribe();
            return rx.recv().await.unwrap();
        }

        // 创建新请求
        let (tx, _) = broadcast::channel(1);
        self.requests.insert(key.to_string(), tx.clone());

        let result = f().await;
        let _ = tx.send(result.clone());
        self.requests.remove(key);

        result
    }
}
```

## 待深入研究
- [x] 数据切片架构：已完成分析
- [x] Singleflight 机制：已完成分析
- [x] 预读算法：已完成分析 (详见 [algorithms.md](../algorithms.md#algorithm-5-预读算法-prefetcher-worker-pool))
- [x] 缓存淘汰：已完成分析 (详见 [algorithms.md](../algorithms.md#algorithm-4-缓存淘汰策略))
- [x] Writeback 模式：已完成分析 (详见 [algorithms.md](../algorithms.md#algorithm-6-writeback-异步上传))
- [x] 分片上传：已完成分析 (详见 [algorithms.md](../algorithms.md#algorithm-7-分片上传-multipart-upload))
