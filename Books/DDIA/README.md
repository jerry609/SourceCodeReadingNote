# DDIA 读书笔记

> **Designing Data-Intensive Applications** - Martin Kleppmann
>
> 数据密集型应用设计，分布式系统领域的经典著作。

## 书籍信息

- **作者**: Martin Kleppmann
- **出版年份**: 2017
- **豆瓣评分**: 9.7
- **核心主题**: 数据系统、分布式系统、一致性、可靠性

## 目录结构

### Part I: Foundations of Data Systems (数据系统基础)

| 章节 | 标题 | 核心概念 |
|:-----|:-----|:---------|
| [Ch1](part1-foundations/ch01-reliable-scalable-maintainable.md) | Reliable, Scalable, and Maintainable Applications | 可靠性、可扩展性、可维护性 |
| [Ch2](part1-foundations/ch02-data-models-query-languages.md) | Data Models and Query Languages | 关系模型、文档模型、图模型 |
| [Ch3](part1-foundations/ch03-storage-and-retrieval.md) | Storage and Retrieval | LSM-Tree、B-Tree、列存储 |
| [Ch4](part1-foundations/ch04-encoding-and-evolution.md) | Encoding and Evolution | 序列化、Schema 演进 |

### Part II: Distributed Data (分布式数据)

| 章节 | 标题 | 核心概念 |
|:-----|:-----|:---------|
| [Ch5](part2-distributed-data/ch05-replication.md) | Replication | 主从复制、多主复制、无主复制 |
| [Ch6](part2-distributed-data/ch06-partitioning.md) | Partitioning | 分片策略、再平衡、路由 |
| [Ch7](part2-distributed-data/ch07-transactions.md) | Transactions | ACID、隔离级别、2PC |
| [Ch8](part2-distributed-data/ch08-trouble-with-distributed.md) | The Trouble with Distributed Systems | 网络分区、时钟、故障模型 |
| [Ch9](part2-distributed-data/ch09-consistency-consensus.md) | Consistency and Consensus | 线性一致性、Raft、Zab |

### Part III: Derived Data (派生数据)

| 章节 | 标题 | 核心概念 |
|:-----|:-----|:---------|
| [Ch10](part3-derived-data/ch10-batch-processing.md) | Batch Processing | MapReduce、Spark、数据流 |
| [Ch11](part3-derived-data/ch11-stream-processing.md) | Stream Processing | 事件溯源、CDC、流处理引擎 |
| [Ch12](part3-derived-data/ch12-future-of-data-systems.md) | The Future of Data Systems | 数据集成、正确性、伦理 |

## 与源码项目的关联

详见 [cross-references.md](cross-references.md) - DDIA 理论概念与实际项目源码的对照。

## 阅读建议

1. **基础篇 (Part I)**: 必读，建立数据系统的基本概念
2. **分布式篇 (Part II)**: 重点章节，理解分布式系统的核心挑战
3. **派生篇 (Part III)**: 了解数据处理的两种范式

## 推荐阅读顺序

```
Ch1 → Ch3 → Ch5 → Ch7 → Ch9 → Ch8 → Ch6 → Ch2 → Ch4 → Ch10 → Ch11 → Ch12
```

**理由**: 先建立整体认知 (Ch1)，再深入存储引擎 (Ch3)，然后理解分布式数据的核心问题 (Ch5-9)。
