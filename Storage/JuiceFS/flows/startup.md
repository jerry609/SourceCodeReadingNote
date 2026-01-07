# JuiceFS 启动流程 (Mount Startup)

本文以 `juicefs mount` 为主线，梳理“客户端是如何把 meta/object/chunk/vfs 组装成一个可工作的文件系统”的。

入口：`main.go` → `cmd.Main` → `cmd/mount.go:mount`

---

## 总览时序

```
juicefs mount META-URL MOUNTPOINT [flags...]
  ├─ setup/prepare mountpoint（含 daemon stage 处理）
  ├─ meta.NewClient(metaURL, metaConf)
  ├─ metaCli.Load(true)            -> 读取 Format（卷名/存储配置/限速等）
  ├─ NewReloadableStorage(format)  -> 初始化 ObjectStorage
  ├─ chunk.NewCachedStore(blob)    -> 初始化 ChunkStore（缓存/压缩/writeback）
  ├─ metaCli.NewSession(true)      -> 建立 session（心跳/锁/可观测性）
  ├─ vfs.NewVFS(vfsConf, metaCli, store)
  ├─ initBackgroundTasks(...)
  └─ mountMain(vfs, c)             -> 启动 FUSE/网关等服务，进入主循环
```

---

## 关键步骤拆解（按代码顺序）

### 1) daemon stage：前台/后台与“多阶段启动”

`cmd/mount.go:mount` 有一个 stage 机制（`__DAEMON_STAGE` / `JFS_SUPERVISOR`），大体目的是：
- stage 0：尽早检查（meta 可连、mountpoint 合法等）并准备 daemon 运行环境
- stage < 3：可能 fork/daemonize（不同平台逻辑不同）
- stage 3：真正提供服务的进程（创建 store/vfs、跑 mountMain）

这使得“启动失败尽早暴露”，同时支持后台守护进程方式运行。

### 2) MetaClient + Format：拿到“卷配置真相之源”

```go
metaCli = meta.NewClient(addr, metaConf)
format, err = metaCli.Load(true)
```

Format 决定了：
- 卷名、对象存储 bucket/参数
- upload/download limit 等运行时参数
- 一些 feature flag（例如是否启用 dir stats 等）

注意：metaConf 还会影响一致性/行为（例如只读、subdir、大小写敏感等）。

### 3) ObjectStorage：把底层后端统一成 `ObjectStorage` 接口

```go
blob, err = NewReloadableStorage(format, metaCli, updateFormat(c))
```

这里会根据 Format 选择并初始化具体后端（S3/OSS/GCS/本地/…），对上提供统一的 Get/Put/List 等接口。

### 4) ChunkStore（CachedStore）：数据面核心

```go
store := chunk.NewCachedStore(blob, *chunkConf, registerer)
```

CachedStore 在启动阶段完成“数据面策略”装配：
- L1/L2 缓存（内存/磁盘）
- singleflight（避免缓存击穿）
- 预读（prefetch）
- writeback（可选：先落本地 staging，后台上传）
- 压缩/解压（可选：LZ4/ZSTD…）

### 5) Session：把客户端纳入元数据后端的“活动集合”

```go
err = metaCli.NewSession(true)
```

Session 一般用于：
- 心跳与存活检测
- 分布式锁/租约（文件锁、POSIX 锁）
- 失败恢复时的清理（例如持有的锁、sustained inode 等）

### 6) VFS：把“文件系统操作”落到 meta + chunk

```go
v := vfs.NewVFS(vfsConf, metaCli, store, registerer, registry)
mountMain(v, c)
```

VFS 负责：
- 文件句柄（handle）生命周期
- Read/Write 的分发（调用 chunk store）
- 元数据操作（调用 meta）
- 挂载入口（FUSE）以及相关的后台任务/监控/信号处理

---

## 你应该从启动链路得到的心智模型

1) **Format 是配置真相之源**：存储后端/限速等从 meta 读取
2) **meta/object/chunk/vfs 是四层组装**：VFS 只依赖接口，不直接依赖具体后端
3) **session 让“分布式语义”成立**：锁/心跳/恢复都需要它

