# Chapter 7: Transactions

> 事务 - 并发控制与一致性保证

## 核心概念

### 7.1 ACID

| 属性 | 英文 | 含义 |
|:-----|:-----|:-----|
| 原子性 | Atomicity | 全部成功或全部失败 |
| 一致性 | Consistency | 约束始终满足 |
| 隔离性 | Isolation | 并发事务互不干扰 |
| 持久性 | Durability | 提交后数据不丢失 |

### 7.2 BASE (NoSQL 的另一种选择)

- **Basically Available**: 基本可用
- **Soft state**: 软状态
- **Eventual consistency**: 最终一致

## 隔离级别

### 并发问题

| 问题 | 描述 |
|:-----|:-----|
| 脏读 | 读取未提交的数据 |
| 脏写 | 覆盖未提交的数据 |
| 不可重复读 | 同一事务内两次读取不一致 |
| 幻读 | 范围查询结果被其他事务改变 |
| 写偏斜 | 两个事务基于相同读取做出冲突决策 |

### 隔离级别对比

| 级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|:-----|:-----|:-----------|:-----|:-----|
| Read Uncommitted | ✗ | ✗ | ✗ | 最高 |
| Read Committed | ✓ | ✗ | ✗ | 高 |
| Repeatable Read | ✓ | ✓ | ✗ | 中 |
| Serializable | ✓ | ✓ | ✓ | 低 |

## 实现机制

### 快照隔离 (Snapshot Isolation)

```
每个事务看到数据库在某个时间点的一致快照
```

**实现方式**: MVCC (多版本并发控制)

```
每行数据有多个版本:
row_v1 (created_by: txn_100, deleted_by: txn_200)
row_v2 (created_by: txn_200, deleted_by: null)

事务只看到 txn_id < 自己 且未被删除的版本
```

### 防止丢失更新

| 策略 | 说明 |
|:-----|:-----|
| 原子操作 | `UPDATE SET x = x + 1` |
| 显式加锁 | `SELECT ... FOR UPDATE` |
| 自动检测 | 检测到冲突则回滚 |
| CAS | Compare-And-Set |

### 写偏斜与幻读

```go
// 医院值班问题
tx1: SELECT count(*) WHERE on_call = true  // 返回 2
tx2: SELECT count(*) WHERE on_call = true  // 返回 2
tx1: UPDATE SET on_call = false WHERE doctor = 'Alice'
tx2: UPDATE SET on_call = false WHERE doctor = 'Bob'
// 结果: 没人值班!
```

**解决方案**:
- 物化冲突: 创建锁定行
- 可序列化隔离

## 可序列化实现

### 1. 真正串行执行

- 单线程执行所有事务
- 存储过程避免网络往返
- 分区提高吞吐

**代表**: Redis, VoltDB

### 2. 两阶段锁 (2PL)

```
规则:
- 读取前获取共享锁
- 写入前获取排他锁
- 持有锁直到事务结束
```

**问题**: 死锁、性能低

### 3. 可序列化快照隔离 (SSI)

```
乐观并发控制:
1. 允许事务并行执行
2. 提交时检测冲突
3. 冲突则回滚重试
```

**代表**: PostgreSQL, CockroachDB

## 分布式事务

### 两阶段提交 (2PC)

```
阶段 1 (Prepare):
  协调者 → 所有参与者: "准备好了吗?"
  参与者: "是" / "否"

阶段 2 (Commit):
  协调者 → 所有参与者: "提交" / "回滚"
```

**问题**:
- 协调者单点故障
- 同步阻塞
- 数据不一致风险

### 三阶段提交 (3PC)

增加 PreCommit 阶段，降低阻塞概率

## 要点总结

1. ACID 中的 I (隔离性) 有多个级别
2. MVCC 是实现快照隔离的主流方式
3. 可序列化有三种实现方式
4. 分布式事务代价高昂

## 思考题

- [ ] 快照隔离和可序列化的区别是什么？
- [ ] 什么场景下会发生写偏斜？
- [ ] 2PC 的协调者故障会怎样？

## 源码关联

| 概念 | 相关项目 | 实现 |
|:-----|:---------|:-----|
| MVCC | TiKV | Percolator 事务模型 |
| 乐观锁 | JuiceFS Redis | WATCH + MULTI |
| 悲观锁 | JuiceFS SQL | SELECT FOR UPDATE |
| 2PC | TiKV | 两阶段提交 |
