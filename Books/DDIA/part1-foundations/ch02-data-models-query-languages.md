# Chapter 2: Data Models and Query Languages

> 数据模型与查询语言

## 核心概念

### 2.1 数据模型的层次

```
应用程序 → 数据模型 (JSON/关系/图) → 存储格式 (字节) → 硬件
```

### 2.2 主流数据模型

| 模型 | 代表 | 适用场景 |
|:-----|:-----|:---------|
| 关系模型 | MySQL, PostgreSQL | 结构化数据、复杂查询 |
| 文档模型 | MongoDB, CouchDB | 半结构化、嵌套数据 |
| 图模型 | Neo4j, JanusGraph | 复杂关系、社交网络 |

## 关系模型 vs 文档模型

### 关系模型

- **优点**: 支持 JOIN、事务、成熟的查询优化器
- **缺点**: 对象-关系阻抗不匹配 (Impedance Mismatch)

### 文档模型

- **优点**: 模式灵活、局部性好、接近应用数据结构
- **缺点**: 不支持 JOIN、多对多关系困难

### 融合趋势

- PostgreSQL 支持 JSON
- MongoDB 支持 $lookup (类似 JOIN)

## 图模型

### 属性图 (Property Graph)

```
顶点 (Vertex): {id, properties, outgoing_edges, incoming_edges}
边 (Edge): {id, properties, head_vertex, tail_vertex, label}
```

### 查询语言

- **Cypher** (Neo4j)
- **Gremlin** (Apache TinkerPop)
- **SPARQL** (RDF)

## 查询语言

### 声明式 vs 命令式

| 类型 | 特点 | 示例 |
|:-----|:-----|:-----|
| 声明式 | 描述目标，不关心实现 | SQL, CSS |
| 命令式 | 描述步骤，控制流程 | JavaScript, Python |

### MapReduce

- 介于声明式和命令式之间
- map 和 reduce 函数必须是纯函数

## 要点总结

1. 数据模型深刻影响软件设计
2. 关系模型适合多对多关系
3. 文档模型适合一对多嵌套结构
4. 图模型适合高度关联数据

## 思考题

- [ ] 你的应用适合哪种数据模型？
- [ ] 何时应该使用嵌套文档 vs 规范化？
- [ ] 图数据库解决了什么问题？

## 源码关联

| 概念 | 相关项目 | 实现 |
|:-----|:---------|:-----|
| 文档存储 | JuiceFS | inode 属性二进制序列化 |
| 层次结构 | JuiceFS | 目录树 (d$inode Hash) |
