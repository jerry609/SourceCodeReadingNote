# pkg/object 模块分析

## 职责
`pkg/object` 是 JuiceFS 的对象存储适配层。它定义了一个通用的 `ObjectStorage` 接口，屏蔽了底层 50+ 种不同对象存储（S3, OSS, GCS, HDFS, 本地磁盘等）的 API 差异。

**源码位置**: `pkg/object/` 目录

## 支持的后端列表

| 类别 | 后端 | 文件 |
|:-----|:-----|:-----|
| **AWS 兼容** | S3, MinIO, Wasabi, DigitalOcean Spaces | `s3.go`, `minio.go`, `wasabi.go`, `space.go` |
| **中国云厂商** | 阿里云 OSS, 腾讯云 COS, 华为云 OBS, 百度 BOS, 金山云 KS3 | `oss.go`, `cos.go`, `obs.go`, `bos.go`, `ks3.go` |
| **国际云厂商** | Google Cloud Storage, Azure Blob, IBM COS | `gs.go`, `azure.go`, `ibmcos.go` |
| **其他云存储** | 七牛云, 又拍云, UCloud, 火山引擎 TOS | `qiniu.go`, `ufile.go`, `tos.go` |
| **分布式存储** | Ceph (RadosGW), Swift, GlusterFS, HDFS | `ceph.go`, `swift.go`, `gluster.go`, `hdfs.go` |
| **本地/协议** | 本地文件系统, SFTP, NFS, WebDAV, CIFS | `file.go`, `sftp.go`, `nfs.go`, `webdav.go`, `cifs.go` |
| **KV 存储** | Redis, TiKV, etcd | `redis.go`, `tikv.go`, `etcd.go` |
| **数据库** | MySQL, PostgreSQL, SQLite | `sql.go`, `sql_mysql.go`, `sql_pg.go`, `sql_sqlite.go` |

## 核心接口

### Object 接口
```go
// pkg/object/interface.go:25-32
type Object interface {
    Key() string
    Size() int64
    Mtime() time.Time
    IsDir() bool
    IsSymlink() bool
    StorageClass() string
}
```

### ObjectStorage 接口
```go
// pkg/object/interface.go:77-112
type ObjectStorage interface {
    // 描述信息
    String() string
    Limits() Limits

    // 基础操作
    Create(ctx context.Context) error
    Get(ctx context.Context, key string, off, limit int64, getters ...AttrGetter) (io.ReadCloser, error)
    Put(ctx context.Context, key string, in io.Reader, getters ...AttrGetter) error
    Copy(ctx context.Context, dst, src string) error
    Delete(ctx context.Context, key string, getters ...AttrGetter) error

    // 元数据操作
    Head(ctx context.Context, key string) (Object, error)
    List(ctx context.Context, prefix, startAfter, token, delimiter string, limit int64, followLink bool) ([]Object, bool, string, error)
    ListAll(ctx context.Context, prefix, marker string, followLink bool) (<-chan Object, error)

    // 分片上传
    CreateMultipartUpload(ctx context.Context, key string) (*MultipartUpload, error)
    UploadPart(ctx context.Context, key string, uploadID string, num int, body []byte) (*Part, error)
    UploadPartCopy(ctx context.Context, key string, uploadID string, num int, srcKey string, off, size int64) (*Part, error)
    AbortUpload(ctx context.Context, key string, uploadID string)
    CompleteUpload(ctx context.Context, key string, uploadID string, parts []*Part) error
    ListUploads(ctx context.Context, marker string) ([]*PendingPart, string, error)
}
```

### Limits 结构体
```go
// pkg/object/interface.go:67-73
type Limits struct {
    IsSupportMultipartUpload bool
    IsSupportUploadPartCopy  bool
    MinPartSize              int
    MaxPartSize              int64
    MaxPartCount             int
}
```

### MultipartUpload 相关
```go
// pkg/object/interface.go:49-59
type MultipartUpload struct {
    MinPartSize int
    MaxCount    int
    UploadID    string
}

type Part struct {
    Num  int
    Size int
    ETag string
}

type PendingPart struct {
    Key      string
    UploadID string
    Created  time.Time
}
```

## 实现模式

### 注册机制
每个后端通过 `init()` 函数注册自己：

```go
// pkg/object/s3.go
func init() {
    Register("s3", newS3)
}

// pkg/object/object_storage.go
var storageClasses = make(map[string]Creator)

func Register(name string, creator Creator) {
    storageClasses[name] = creator
}
```

### S3 适配器示例
```go
// pkg/object/s3.go (简化)
type s3client struct {
    bucket string
    region string
    client *s3.Client
}

func (s *s3client) Get(ctx context.Context, key string, off, limit int64, getters ...AttrGetter) (io.ReadCloser, error) {
    params := &s3.GetObjectInput{
        Bucket: aws.String(s.bucket),
        Key:    aws.String(key),
    }
    if off > 0 || limit > 0 {
        rng := fmt.Sprintf("bytes=%d-", off)
        if limit > 0 {
            rng = fmt.Sprintf("bytes=%d-%d", off, off+limit-1)
        }
        params.Range = aws.String(rng)
    }
    resp, err := s.client.GetObject(ctx, params)
    return resp.Body, err
}

func (s *s3client) Put(ctx context.Context, key string, in io.Reader, getters ...AttrGetter) error {
    _, err := s.client.PutObject(ctx, &s3.PutObjectInput{
        Bucket: aws.String(s.bucket),
        Key:    aws.String(key),
        Body:   in,
    })
    return err
}
```

## 功能增强层

### 加密层 (encrypted)
```go
// pkg/object/encrypt.go
type encrypted struct {
    ObjectStorage
    key []byte
}

func (e *encrypted) Get(ctx context.Context, key string, off, limit int64, getters ...AttrGetter) (io.ReadCloser, error) {
    // 读取加密数据
    r, err := e.ObjectStorage.Get(ctx, key, 0, 0, getters...)
    // 解密...
    return decryptedReader, nil
}
```

### 校验层 (checksum)
```go
// pkg/object/checksum.go
type checksumStorage struct {
    ObjectStorage
}

func (c *checksumStorage) Put(ctx context.Context, key string, in io.Reader, getters ...AttrGetter) error {
    // 计算校验和
    // 存储数据 + 校验和
}
```

### 前缀包装 (withPrefix)
```go
// pkg/object/prefix.go
type withPrefix struct {
    os     ObjectStorage
    prefix string
}

func (w *withPrefix) Get(ctx context.Context, key string, off, limit int64, getters ...AttrGetter) (io.ReadCloser, error) {
    return w.os.Get(ctx, w.prefix+key, off, limit, getters...)
}
```

### 分片存储 (sharded)
```go
// pkg/object/sharding.go
type sharded struct {
    stores []ObjectStorage
}

func (s *sharded) Get(ctx context.Context, key string, off, limit int64, getters ...AttrGetter) (io.ReadCloser, error) {
    idx := s.shardIndex(key)
    return s.stores[idx].Get(ctx, key, off, limit, getters...)
}
```

## Rust 重写方案：OpenDAL

在 Rust 重写中，**强烈建议直接使用 [Apache OpenDAL](https://github.com/apache/opendal)**，而不是复刻 `pkg/object`。

### 为什么选择 OpenDAL？

| 特性 | JuiceFS pkg/object | OpenDAL |
|:-----|:-------------------|:--------|
| 后端数量 | 50+ | 50+ |
| 语言 | Go | Rust (原生) |
| 异步支持 | 无 | 全异步 |
| 中间件 | 手写 | 内置 Layer 系统 |
| 流式读写 | 部分 | 完整支持 |
| 零拷贝 | 困难 | 设计友好 |
| 社区 | JuiceData | Apache 基金会 |

### OpenDAL Layers (中间件)

```rust
use opendal::{Operator, layers};

let op = Operator::new(S3::default())?
    .layer(layers::LoggingLayer::default())     // 日志
    .layer(layers::RetryLayer::new())           // 重试
    .layer(layers::MetricsLayer::default())     // 指标
    .layer(layers::TracingLayer::default())     // 追踪
    .layer(layers::TimeoutLayer::new()          // 超时
        .with_timeout(Duration::from_secs(30)))
    .finish();
```

### 代码映射
| JuiceFS Go | Rust + OpenDAL |
|:-----------|:---------------|
| `pkg/object/interface.go` | `opendal::Operator` |
| `blob.NewReader` | `op.reader_with(path).range(..).await` |
| `blob.Put` | `op.write(path, bytes).await` |
| `Register("s3", ...)` | `Scheme::S3` builder |
| `MultipartUpload` | `op.writer_with(path).await` (自动处理) |

### 示例代码 (Rust)
```rust
use opendal::{Operator, Scheme, services::S3};

async fn init_object_storage(cfg: &Config) -> Result<Operator> {
    let mut builder = S3::default();
    builder.bucket(&cfg.bucket);
    builder.access_key_id(&cfg.access_key);
    builder.secret_access_key(&cfg.secret_key);
    builder.region(&cfg.region);
    builder.endpoint(&cfg.endpoint);

    let op = Operator::new(builder)?
        .layer(opendal::layers::LoggingLayer::default())
        .layer(opendal::layers::RetryLayer::new()
            .with_max_times(3)
            .with_jitter())
        .finish();

    Ok(op)
}

// 读取
async fn get_object(op: &Operator, key: &str, off: u64, len: u64) -> Result<Vec<u8>> {
    let reader = op.reader_with(key)
        .range(off..off+len)
        .await?;
    reader.read(..).await
}

// 写入 (自动处理分片上传)
async fn put_object(op: &Operator, key: &str, data: Vec<u8>) -> Result<()> {
    op.write(key, data).await
}

// 流式写入
async fn put_object_stream(op: &Operator, key: &str, stream: impl Stream<Item = Bytes>) -> Result<()> {
    let mut writer = op.writer(key).await?;
    pin_mut!(stream);
    while let Some(chunk) = stream.next().await {
        writer.write(chunk).await?;
    }
    writer.close().await
}
```

### 多后端支持
```rust
async fn create_storage(scheme: &str, config: &HashMap<String, String>) -> Result<Operator> {
    match scheme {
        "s3" => {
            let builder = S3::default()
                .bucket(&config["bucket"])
                .access_key_id(&config["access_key"])
                .secret_access_key(&config["secret_key"]);
            Operator::new(builder)?.finish()
        }
        "fs" => {
            let builder = Fs::default().root(&config["root"]);
            Operator::new(builder)?.finish()
        }
        "memory" => {
            Operator::new(Memory::default())?.finish()
        }
        _ => Err(anyhow!("unsupported scheme: {}", scheme))
    }
}
```

## 待深入研究
- [x] 接口设计：已完成分析
- [x] 后端列表：已完成统计
- [ ] 分片上传策略：不同后端的 Part Size 限制
- [ ] 重试机制：各后端的错误处理和重试逻辑
- [ ] 性能优化：连接池、并发控制
