# pkg/object 模块分析

## 职责
`pkg/object` 是 JuiceFS 的对象存储适配层。它定义了一个通用的 `ObjectStorage` 接口，屏蔽了底层几十种不同对象存储（S3, OSS, GCS, HDFS, 本地磁盘等）的 API 差异。

## 核心接口 (Interface)
位于 `pkg/object/interface.go`：
```go
type ObjectStorage interface {
    Get(ctx context.Context, key string, off, limit int64) (io.ReadCloser, error)
    Put(ctx context.Context, key string, in io.Reader) error
    Delete(ctx context.Context, key string) error
    List(ctx context.Context, prefix, marker string, limit int64) ([]Object, bool, string, error)
    // 分片上传相关
    CreateMultipartUpload(...)
    UploadPart(...)
    CompleteUpload(...)
}
```

## 实现模式：S3 适配器
以 `pkg/object/s3.go` 为例：
1.  **依赖**: 引入 AWS Go SDK v2。
2.  **工厂模式**: 使用 `Register("s3", newS3)` 注册构造函数。
3.  **胶水代码**: 处理复杂的 Endpoint 解析（Path-style vs Virtual-host style）、Region 自动探测、Credential 加载。

## Rust 重写方案：OpenDAL

在 Rust 重写中，**强烈建议直接使用 [Apache OpenDAL](https://github.com/apache/opendal)**，而不是复刻 `pkg/object`。

### 为什么选择 OpenDAL？
1.  **开箱即用**: OpenDAL 已经适配了几乎所有 JuiceFS 支持的后端（S3, GCS, Azblob, HDFS, Redis, Moka 等）。
2.  **统一抽象**: OpenDAL 的 `Operator` 结构体提供了比 JuiceFS `ObjectStorage` 更现代、更强大的 API（全异步、流式读写）。
3.  **中间件支持 (Layers)**: OpenDAL 提供了 Retry, Logging, Metrics, Tracing, Timeout 等中间件，这意味着你不需要在业务逻辑里手写这些代码（JuiceFS 的 `chunk` 模块里手写了很多重试逻辑，用 OpenDAL 可以直接省去）。
4.  **零拷贝**: OpenDAL 设计上对 Zero-copy 更友好。

### 代码映射
| JuiceFS Go | Rust + OpenDAL |
| :--- | :--- |
| `pkg/object/interface.go` | `opendal::Operator` |
| `blob.NewReader` | `op.reader_with(path).range(..).await` |
| `blob.Put` | `op.write(path, bytes).await` |
| `Register("s3", ...)` | `Scheme::S3` builder |

### 示例代码 (Rust)
```rust
use opendal::{Operator, Scheme};

async fn init_object_storage(cfg: &Config) -> Result<Operator> {
    let mut builder = opendal::services::S3::default();
    builder.bucket(&cfg.bucket);
    builder.access_key_id(&cfg.access_key);
    builder.secret_access_key(&cfg.secret_key);
    
    let op = Operator::new(builder)?
        .layer(opendal::layers::LoggingLayer::default())
        .layer(opendal::layers::RetryLayer::new())
        .finish();
        
    Ok(op)
}
```
