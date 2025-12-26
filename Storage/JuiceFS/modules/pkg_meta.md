# pkg/meta 模块分析

## 职责
`pkg/meta` 是 JuiceFS 的心脏，负责管理所有的文件系统元数据。它定义了统一的 `Meta` 接口，并实现了多种数据库后端（Redis, SQL, TiKV 等）。

## 核心接口 (Trait)
在 `pkg/meta/interface.go` 中定义的 `Meta` 接口非常庞大：
- **节点操作**: `Mknod`, `Mkdir`, `Lookup`, `GetAttr`, `SetAttr`
- **目录操作**: `Readdir`, `Rename`, `Unlink`
- **数据映射**: `NewSlice`, `Write` (元数据层面), `Read` (获取 Slice 列表)
- **锁与会话**: `Flock`, `Setlk`, `NewSession`

## 核心实现：Redis 后端
Redis 实现 (`pkg/meta/redis.go`) 是 JuiceFS 最常用且最高效的后端之一。

### Redis 数据布局 (Schema)
| 资源类型 | Redis 结构 | Key 格式 | Value / Field 示例 |
| :--- | :--- | :--- | :--- |
| **目录项 (Dentry)** | Hash | `d$inode` | Field: `name`, Value: `[type, inode_id]` |
| **属性 (Attribute)** | String | `i$inode` | 序列化的二进制 `Attr` |
| **符号链接** | String | `s$inode` | 目标路径字符串 |
| **数据分块 (Chunk)** | List/Set | `c$inode_$indx` | 存储 Slice 列表 |

## 关键流程：Lookup
`Lookup` 负责根据父目录 inode 和文件名找到目标的 inode 和属性。

### 优化机制：Lua 脚本
为了减少网络 RTT，JuiceFS 使用 Lua 脚本将 `HGET` (查目录) 和 `GET` (查属性) 合并为一个原子操作。
```lua
-- scriptLookup 逻辑伪代码
local inode = redis.call('HGET', KEYS[1], ARGV[1])
if inode then
    local attr = redis.call('GET', 'i' .. parse_inode(inode))
    return {inode, attr}
end
return nil
```

## Rust 重写思考
1. **Trait 拆分**: 不要实现一个像 Go 那么大的接口。建议拆分为 `MetadataRead`, `MetadataWrite`, `LockManager` 等多个 Trait。
2. **异步支持**: 所有的数据库操作都应该是 `async` 的。
3. **序列化**: 使用 `serde` 和 `bincode` 可以完美模拟 Go 的二进制 `Marshal` 过程。
4. **Lua 脚本管理**: 使用 Rust 的 `redis::Script` 可以很方便地加载和调用这些脚本。

## 待深入研究
- [ ] 事务保证：Redis 的 `Watch` + `Multi` 如何在分布式环境下保证元数据一致性？
- [ ] 锁机制：`Flock` 在 Redis 后端是如何实现的？
