# 快速建立心智模型 (Reading Guide)

这份阅读指南的目标：用最短路径理解 JuiceFS “数据怎么走、元数据怎么走、在哪收敛一致性”。

---

## 0. 项目身份（30 秒）

一句话：**JuiceFS 是“重客户端”的分布式 POSIX 文件系统** —— 客户端把文件系统操作拆成两条链路：
- **元数据链路**：`pkg/meta`（Redis/SQL/TiKV…）负责 inode/dentry/锁/配额…
- **数据链路**：`pkg/chunk` + `pkg/object` 负责切片、缓存、压缩、上传/下载对象存储

你要先回答 3 个问题：
1) 写入的数据什么时候“算成功”？（对象存储成功后才 commit 元数据）
2) 读路径靠什么加速？（多级缓存 + 预读 + singleflight）
3) 一致性靠什么保证/牺牲？（事务/原子脚本 + TTL 缓存 + 可选的 notify/Tracking）

---

## 1. 架构骨架（5 分钟）

先扫目录，建立“模块 → 职责”的映射：
- `main.go` / `cmd/`：CLI 入口（`juicefs mount/format/...`），mount 启动链路在 `cmd/mount.go`
- `pkg/vfs/`：把 FUSE/Hadoop/S3 网关的文件系统操作落到 meta + chunk
- `pkg/meta/`：元数据引擎与后端实现（Redis/SQL/TiKV），以及事务/脚本/锁
- `pkg/chunk/`：数据分块（Chunk/Block/Page）、缓存、压缩、上传/下载
- `pkg/object/`：对象存储适配层（S3/OSS/GCS/… 50+）

读分布式文件系统时的三连问：
1) **元数据如何组织与原子化？**
2) **数据如何分块、命名、回收？**
3) **缓存在哪些层？一致性怎么做？**

---

## 2. 追数据流（10 分钟）

把 JuiceFS 当成两条“管道”：

```
VFS/FS call
  ├─ 元数据：Meta.Lookup/Create/Rename/...
  └─ 数据：ChunkStore.Read/Write -> ObjectStorage.Get/Put
```

建议按频率优先级读三条主路径：
- Lookup：`Storage/JuiceFS/flows/lookup.md`
- Read：`Storage/JuiceFS/flows/read.md`
- Write：`Storage/JuiceFS/flows/write.md`

再补一条“启动链路”，把对象是怎么被拼起来的看清楚：
- Startup：`Storage/JuiceFS/flows/startup.md`

---

## 3. 锚点文件（80/20）

理解这些文件基本就能“跑通大脑里的调用链”：

**启动与装配**
- `cmd/mount.go:mount`：读取 meta format → 构造 object storage → 构造 chunk store → NewSession → NewVFS → mountMain

**VFS 入口**
- `pkg/vfs/vfs.go` / `pkg/vfs/handle.go`：句柄、读写分发、热升级状态恢复

**数据层核心**
- `pkg/chunk/cached_store.go`：读写缓存、writeback、singleflight、prefetch

**元数据核心**
- `pkg/meta/interface.go`：`Meta` 接口（契约）
- `pkg/meta/redis.go`：Redis 实现（事务/脚本/可选 client-side caching）
- `pkg/meta/tkv.go` + `pkg/meta/tkv_tikv.go`：TiKV 实现（事务 + MVCC）

---

## 4. 一致性与不变量（先抓住边界）

最值得先记住的 3 个“硬约束”：
1) **数据先落对象存储，元数据后 commit**（避免“元数据指向不存在的数据”）
2) **元数据更新要么原子，要么可重试**（Redis 事务/Lua、TiKV 事务 + retry）
3) **跨客户端缓存一致性通常是“TTL + 局部失效”**（强一致需要额外机制，例如 Redis CLIENT TRACKING）

相关问答入口：`Storage/JuiceFS/questions.md`

---

## 5. 最小验证（读完就去做）

读完主链路后，用一个最小实验验证你的理解：
1) 跟一次 `cmd/mount.go:mount` 看启动时装配了哪些组件（meta/store/vfs）
2) 在 `pkg/chunk/cached_store.go` 里追一次 Read Miss（singleflight + 下载 + 写缓存）
3) 在 `pkg/meta/redis.go` 里追一次涉及多 key 的更新（`txn()` + Watch + TxPipelined）

