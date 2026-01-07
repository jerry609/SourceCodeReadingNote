# 权衡取舍 (Tradeoffs)

> 工程没有银弹，只有取舍。
> 记录项目中做出的关键设计决策，分析其利弊和适用场景。

## 常见权衡维度

- **一致性 vs 可用性** (CAP)
- **性能 vs 简单性**
- **灵活性 vs 性能**
- **开发效率 vs 运行效率**
- **抽象程度 vs 可理解性**

---

## Tradeoff 1: 统一 TrainJob vs 多 CRD

### 场景
如何设计用户 API？是为每种 ML 框架设计独立 CRD，还是使用统一的 TrainJob？

### 选项分析

#### 选项 A: 多 CRD (Training Operator V1 方案)
```yaml
# PyTorchJob
apiVersion: kubeflow.org/v1
kind: PyTorchJob
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
    Worker:
      replicas: 4
---
# MPIJob
apiVersion: kubeflow.org/v1
kind: MPIJob
spec:
  slotsPerWorker: 8
  mpiReplicaSpecs:
    Launcher: ...
    Worker: ...
```

**优点**:
- 每种框架的 API 可以针对性设计
- 类型安全，框架特定字段有明确定义
- 用户可以只安装需要的 Operator

**缺点**:
- 代码重复严重 (每种框架一个 Controller)
- 功能同步困难
- 用户需要学习多种 CRD
- 维护成本高

**适用场景**: 框架间差异极大，难以抽象

#### 选项 B: 统一 TrainJob (Trainer V2 方案)
```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
spec:
  runtimeRef:
    name: torch-distributed  # 或 mpi-distributed
  trainer:
    numNodes: 4
    numProcPerNode: 8
```

**优点**:
- 统一 API，降低学习成本
- 代码复用，通过插件扩展
- 运维简单，只需部署一个 Controller
- 易于添加新框架支持

**缺点**:
- API 抽象可能不够精确
- 框架特定功能可能难以表达
- 需要更复杂的插件机制

**适用场景**: 框架间有共同抽象，可以统一建模

### 项目选择
Trainer V2 选择了 **选项 B: 统一 TrainJob**

### 选择理由
1. 分布式训练的核心概念 (节点、进程、调度) 是通用的
2. 框架特定配置可以通过 `mlPolicy` 字段和插件处理
3. 统一 API 对用户更友好
4. 便于集成 Kubeflow SDK

### 代价
- 需要设计良好的插件系统
- 某些框架特定功能可能需要妥协

### 缓解措施
- 通过 `podTemplateOverrides` 提供灵活的定制能力
- 插件可以注入任意环境变量和配置

---

## Tradeoff 2: JobSet 依赖 vs 自定义工作负载

### 场景
如何表达训练任务的工作负载？是依赖外部 JobSet，还是自己实现工作负载控制器？

### 选项分析

#### 选项 A: 依赖 JobSet
**优点**:
- 复用成熟的工作负载管理能力
- JobSet 提供了 startup/shutdown 顺序控制
- 减少代码量和维护负担
- 与 Kubernetes 生态更好集成

**缺点**:
- 增加外部依赖
- JobSet 的 bug 会影响 Trainer
- 某些功能可能受限于 JobSet 能力

#### 选项 B: 自定义工作负载
**优点**:
- 完全控制工作负载行为
- 可以针对训练场景优化
- 无外部依赖

**缺点**:
- 需要实现复杂的 Pod 管理逻辑
- 重复造轮子
- 维护成本高

### 项目选择
Trainer 选择了 **选项 A: 依赖 JobSet**

### 选择理由
1. JobSet 是 Kubernetes SIG-Apps 维护的项目，质量有保证
2. JobSet 的能力 (replicated jobs, startup order) 非常适合训练场景
3. 专注于训练相关逻辑，而非通用工作负载管理

### 代价
- 必须预先安装 JobSet CRD
- 某些功能 (如弹性训练) 需要等待 JobSet 支持

### 缓解措施
- 在文档中明确依赖要求
- 积极参与 JobSet 社区，推动需要的功能

---

## Tradeoff 3: 插件静态注册 vs 动态加载

### 场景
插件如何注册到框架？是编译时静态注册，还是运行时动态加载？

### 选项分析

#### 选项 A: 静态注册 (当前方案)
```go
func NewRegistry() Registry {
    return Registry{
        torch.Name:        torch.New,
        mpi.Name:          mpi.New,
        jobset.Name:       jobset.New,
    }
}
```

**优点**:
- 简单直接
- 编译时类型检查
- 无运行时开销
- 依赖清晰

**缺点**:
- 添加/移除插件需要重新编译
- 无法在不同部署中启用不同插件

#### 选项 B: 动态加载 (配置文件)
```yaml
# 类似 Kubernetes Scheduler
plugins:
  enabledPlugins:
    - torch
    - mpi
  disabledPlugins:
    - volcano
```

**优点**:
- 运行时可配置
- 不同环境可使用不同插件集
- 便于 A/B 测试

**缺点**:
- 增加运行时复杂性
- 可能出现配置错误
- 需要处理插件依赖

### 项目选择
Trainer 选择了 **选项 A: 静态注册**

### 选择理由
1. 项目仍在 alpha 阶段，快速迭代比配置灵活性更重要
2. 插件数量有限，静态注册足够
3. 简化部署和调试

### 代价
- 无法按需启用/禁用插件
- 需要维护多个构建配置

### 缓解措施
- 预留了扩展点，未来可以添加配置支持
- 插件设计为无副作用，不匹配时自动跳过

---

## Tradeoff 4: Server-Side Apply vs Client-Side Apply

### 场景
如何更新 Kubernetes 资源？使用 Server-Side Apply 还是传统的 Create/Update？

### 选项分析

#### 选项 A: Server-Side Apply (当前方案)
**优点**:
- 声明式，只描述期望状态
- 自动处理冲突
- 多 Controller 共存
- 类型安全

**缺点**:
- 需要 Kubernetes 1.22+
- 学习曲线
- Debug 较复杂

#### 选项 B: Client-Side Apply (Create/Update)
**优点**:
- 传统模式，文档丰富
- 支持旧版 Kubernetes
- 行为直观

**缺点**:
- 需要手动处理冲突
- 容易覆盖其他 Controller 的修改
- 代码繁琐

### 项目选择
Trainer 选择了 **选项 A: Server-Side Apply**

### 选择理由
1. Kubernetes 1.22+ 已广泛部署
2. SSA 更符合 Kubernetes 声明式理念
3. 与其他 Controller (如 Kueue) 更好共存

### 代价
- 不支持老版本 Kubernetes
- 开发者需要学习 Apply Configuration API

---

## 决策矩阵

| 决策点 | 选择 | 放弃 | 核心考量 | 代价 |
|:-------|:-----|:-----|:---------|:-----|
| CRD 设计 | 统一 TrainJob | 多框架独立 CRD | 用户体验、可维护性 | 需要复杂插件系统 |
| 工作负载 | JobSet 依赖 | 自定义工作负载 | 复用、专注 | 外部依赖 |
| 插件注册 | 静态注册 | 动态加载 | 简单性 | 灵活性受限 |
| 资源更新 | SSA | CRUD | 声明式、冲突处理 | 版本要求 |

## 总结思考

Trainer 的设计决策体现了以下原则：

1. **用户优先**: 统一 API 降低用户学习成本
2. **复用优于重造**: 依赖 JobSet 等成熟组件
3. **简单优先**: 在满足需求的前提下选择最简单的方案
4. **渐进演化**: 预留扩展点，但不过度设计
