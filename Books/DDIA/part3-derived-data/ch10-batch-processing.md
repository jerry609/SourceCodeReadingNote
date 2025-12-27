# Chapter 10: Batch Processing

> 批处理 - 大规模离线数据处理

## 核心概念

### 10.1 三种处理系统

| 类型 | 输入 | 输出 | 延迟 |
|:-----|:-----|:-----|:-----|
| 服务 (在线) | 请求 | 响应 | 毫秒级 |
| 批处理 (离线) | 有界数据集 | 派生数据集 | 分钟~小时 |
| 流处理 (近实时) | 无界数据流 | 派生数据/事件 | 秒级 |

### 10.2 Unix 哲学

```bash
cat /var/log/nginx/access.log |
  awk '{print $7}' |
  sort |
  uniq -c |
  sort -rn |
  head -n 5
```

**核心原则**:
- 每个程序做一件事
- 输出可作为另一个程序的输入
- 文本是通用接口

## MapReduce

### 编程模型

```
输入 → Map → Shuffle → Reduce → 输出

Map: (k1, v1) → list(k2, v2)
Reduce: (k2, list(v2)) → list(v3)
```

### 执行流程

```
1. 分片输入数据
2. 每个 Mapper 处理一个分片
3. Shuffle: 按 key 分组发送到 Reducer
4. Reducer 聚合输出
```

### Join 实现

| 类型 | 实现方式 | 适用场景 |
|:-----|:---------|:---------|
| Sort-Merge Join | 两边按 key 排序后合并 | 两个大表 |
| Broadcast Hash Join | 小表广播到所有节点 | 一大一小 |
| Partitioned Hash Join | 两边按 key 分区 | 两个大表 |

### 工作流调度

- 多个 MapReduce 作业组成 DAG
- 调度器: Oozie, Azkaban, Airflow

## 超越 MapReduce

### MapReduce 的问题

- 中间结果写磁盘 (I/O 开销大)
- 多轮作业需要协调
- 编程模型受限

### 数据流引擎

| 系统 | 特点 |
|:-----|:-----|
| Spark | RDD 内存计算 |
| Flink | 流批一体 |
| Tez | DAG 执行引擎 |

### 改进点

- 中间结果可以在内存传递
- 算子可以流水线执行
- 更丰富的算子 (不只是 Map/Reduce)

## 高级 API

### 声明式查询

```sql
-- Hive / Spark SQL
SELECT user_id, COUNT(*) as cnt
FROM events
WHERE event_type = 'click'
GROUP BY user_id
```

**优势**:
- 查询优化器
- 更少的代码
- 更好的可读性

### 图处理

- **Pregel 模型**: 顶点间消息传递
- **系统**: GraphX (Spark), Giraph

## 要点总结

1. MapReduce 是批处理的基础抽象
2. 数据流引擎比 MapReduce 更高效
3. 声明式 API 简化开发并允许优化
4. 批处理适合数据仓库、ETL、机器学习训练

## 思考题

- [ ] MapReduce 的 Shuffle 阶段做了什么？
- [ ] 为什么 Spark 比 MapReduce 快？
- [ ] 什么场景适合批处理？

## 源码关联

| 概念 | 相关项目 | 实现 |
|:-----|:---------|:-----|
| 批量处理 | JuiceFS | `juicefs gc --compact` |
| 并行扫描 | JuiceFS | 多线程元数据遍历 |
| ETL | JuiceFS | `juicefs sync` 数据同步 |
