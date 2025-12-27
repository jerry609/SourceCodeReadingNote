# Chapter 11: Stream Processing

> 流处理 - 实时数据处理

## 核心概念

### 11.1 流 vs 批

| 特性 | 批处理 | 流处理 |
|:-----|:-------|:-------|
| 数据 | 有界 | 无界 |
| 延迟 | 分钟~小时 | 秒~毫秒 |
| 结果 | 一次计算 | 持续更新 |

### 11.2 事件流

```
事件 = (timestamp, key, value)

Producer → Message Broker → Consumer
```

## 消息系统

### 直接消息传递

- UDP 多播
- ZeroMQ
- 问题: 生产者比消费者快怎么办？

### 消息代理 (Message Broker)

| 类型 | 代表 | 特点 |
|:-----|:-----|:-----|
| 传统队列 | RabbitMQ, ActiveMQ | ACK 后删除，负载均衡 |
| 日志式 | Kafka, Pulsar | 持久化，可重放 |

### Kafka 架构

```
Topic → Partition 0 → [msg0, msg1, msg2, ...]
      → Partition 1 → [msg0, msg1, msg2, ...]
      → Partition 2 → [msg0, msg1, msg2, ...]

Consumer Group: 每个分区只被组内一个消费者消费
```

## 数据库与流

### 变更数据捕获 (CDC)

```
数据库写入 → 捕获变更 → 发布到消息队列

方式:
- 触发器
- 解析 binlog
- 逻辑复制
```

### 事件溯源 (Event Sourcing)

```
传统: 存储当前状态
事件溯源: 存储所有事件，状态由事件推导

Event 1: AccountCreated(user_id=1, balance=0)
Event 2: Deposited(user_id=1, amount=100)
Event 3: Withdrawn(user_id=1, amount=30)

当前状态: balance = 70
```

**优势**:
- 完整审计日志
- 可以重建任意时间点状态
- 支持时态查询

## 流处理

### 处理模式

| 模式 | 说明 |
|:-----|:-----|
| 简单转换 | map, filter |
| 聚合 | count, sum, avg (需要窗口) |
| Join | 流-流 Join, 流-表 Join |
| CEP | 复杂事件处理，模式匹配 |

### 时间语义

| 类型 | 定义 |
|:-----|:-----|
| 事件时间 | 事件实际发生的时间 |
| 处理时间 | 系统处理事件的时间 |
| 摄入时间 | 进入系统的时间 |

### 窗口类型

```
滚动窗口 (Tumbling): [0-10) [10-20) [20-30)
滑动窗口 (Sliding):  [0-10) [5-15) [10-20)
会话窗口 (Session):  按活动间隔分组
```

### 水位线 (Watermark)

```
Watermark(t) = "不会再有 event_time < t 的事件"

用于触发窗口计算
处理迟到数据: 允许一定延迟，超时丢弃或侧输出
```

## 流处理引擎

| 系统 | 特点 |
|:-----|:-----|
| Kafka Streams | 轻量级库 |
| Apache Flink | 低延迟，精确一次 |
| Apache Spark Streaming | 微批处理 |
| Apache Storm | 早期流处理 |

### 容错机制

| 语义 | 说明 |
|:-----|:-----|
| At-most-once | 可能丢失 |
| At-least-once | 可能重复 |
| Exactly-once | 精确一次 (需要幂等或事务) |

## 要点总结

1. 消息代理解耦生产者和消费者
2. CDC 将数据库变更转为事件流
3. 事件溯源以事件为核心而非状态
4. 流处理需要处理时间和窗口

## 思考题

- [ ] Kafka 如何保证消息顺序？
- [ ] 事件时间和处理时间的区别是什么？
- [ ] 如何实现 Exactly-once 语义？

## 源码关联

| 概念 | 相关项目 | 实现 |
|:-----|:---------|:-----|
| 变更通知 | JuiceFS | 文件系统事件监听 |
| 日志 | JuiceFS | Access Log |
| 异步处理 | JuiceFS | Writeback 模式 |
