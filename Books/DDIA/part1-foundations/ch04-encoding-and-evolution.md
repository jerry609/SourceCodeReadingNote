# Chapter 4: Encoding and Evolution

> 编码与演化 - 数据格式与兼容性

## 核心概念

### 4.1 编码格式分类

| 类型 | 示例 | 特点 |
|:-----|:-----|:-----|
| 语言特定 | Java Serializable, Python pickle | 跨语言困难 |
| 文本格式 | JSON, XML, CSV | 人类可读，体积大 |
| 二进制格式 | Protocol Buffers, Avro, Thrift | 紧凑高效 |

### 4.2 兼容性类型

| 类型 | 定义 |
|:-----|:-----|
| 向后兼容 | 新代码能读取旧数据 |
| 向前兼容 | 旧代码能读取新数据 |

## 文本格式的问题

### JSON/XML 的限制

- 数字精度问题 (JSON 大整数)
- 不支持二进制数据 (需要 Base64)
- Schema 可选，类型不明确

## 二进制编码

### Protocol Buffers

```protobuf
message Person {
  required string name = 1;
  optional int32 age = 2;
  repeated string emails = 3;
}
```

**编码方式**: field_tag + type + value

### Avro

```json
{
  "type": "record",
  "name": "Person",
  "fields": [
    {"name": "name", "type": "string"},
    {"name": "age", "type": ["null", "int"], "default": null}
  ]
}
```

**特点**:
- 不存储字段标签，更紧凑
- 需要 Writer Schema 和 Reader Schema

### Thrift

两种编码格式：
- **BinaryProtocol**: 类似 Protobuf
- **CompactProtocol**: 更紧凑

## Schema 演化

### Protobuf/Thrift 兼容规则

| 操作 | 向后兼容 | 向前兼容 |
|:-----|:---------|:---------|
| 添加 optional 字段 | ✓ | ✓ |
| 删除 optional 字段 | ✓ | ✓ |
| 添加 required 字段 | ✗ | ✓ |
| 修改字段类型 | 取决于类型 | 取决于类型 |

### Avro 兼容规则

- 添加/删除带默认值的字段是安全的
- Reader Schema 和 Writer Schema 需要兼容

## 数据流模式

### 1. 数据库

- 应用程序写入数据库
- 旧版本应用可能读取新版本写入的数据

### 2. REST/RPC

- 服务端和客户端版本可能不同
- API 版本管理策略

### 3. 消息队列

- 生产者和消费者独立演化
- Schema Registry 管理版本

## 要点总结

1. 二进制编码比 JSON 更紧凑高效
2. Schema 演化需要考虑双向兼容
3. 不同数据流模式有不同的兼容性挑战
4. Avro 适合动态生成 Schema 的场景

## 思考题

- [ ] 什么时候选择 JSON，什么时候选择 Protobuf？
- [ ] 如何安全地演化 Schema？
- [ ] 微服务间通信应该用什么编码格式？

## 源码关联

| 概念 | 相关项目 | 实现 |
|:-----|:---------|:-----|
| 二进制编码 | JuiceFS | Attr 结构体二进制序列化 |
| Schema 演化 | JuiceFS | 元数据格式版本兼容 |
| RPC | JuiceFS Gateway | gRPC 服务接口 |
