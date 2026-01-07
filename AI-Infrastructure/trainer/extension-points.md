# 扩展点设计 (Extension Points)

> 把变化安置在"插件边缘"，把稳定留在"内核中心"。

## 设计原则

1. **识别变化**: ML 框架多样（Torch/MPI/DeepSpeed）、调度器多样（默认/Volcano/Coscheduling）
2. **封装变化**: 做成可插拔的 Plugin
3. **保护核心**: Controller 和 Runtime 接口保持稳定

---

## 架构图

```
                    ┌─────────────────────────────────────┐
                    │         Core (稳定)                 │
                    │  ┌─────────────────────────────────┐│
                    │  │    TrainJob Controller          ││
                    │  │    Runtime Interface            ││
                    │  │    Framework Interface          ││
                    │  └─────────────────────────────────┘│
                    └─────────────────────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         ▼                           ▼                           ▼
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│  MLPolicy       │       │  Customization  │       │  ComponentBuilder│
│  Plugins        │       │  Plugins        │       │  Plugins        │
│  ┌───────────┐  │       │  ┌───────────┐  │       │  ┌───────────┐  │
│  │   Torch   │  │       │  │  JobSet   │  │       │  │  JobSet   │  │
│  │   MPI     │  │       │  │Coscheduling│ │       │  │  Builder  │  │
│  │ DeepSpeed │  │       │  │  Volcano  │  │       │  └───────────┘  │
│  └───────────┘  │       │  └───────────┘  │       └─────────────────┘
└─────────────────┘       └─────────────────┘
```

---

## 扩展点清单

| 扩展点 | 类型 | 变化频率 | 隔离机制 | 源码位置 |
|:-------|:-----|:---------|:---------|:---------|
| EnforceMLPolicy | Plugin | 中 | 接口契约 | `pkg/runtime/framework/interface.go` |
| EnforceCustomization | Plugin | 中 | 接口契约 | `pkg/runtime/framework/interface.go` |
| ComponentBuilder | Plugin | 低 | 接口契约 | `pkg/runtime/framework/interface.go` |
| Webhook (Validate) | Plugin | 中 | 接口契约 | `pkg/runtime/framework/interface.go` |
| Runtime | Interface | 低 | 接口契约 | `pkg/runtime/interface.go` |

---

## Extension Point 1: EnforceMLPolicy

### 设计意图

不同 ML 框架（PyTorch、MPI、DeepSpeed）需要不同的配置注入方式：
- PyTorch: 设置 `torchrun` 命令和 `PET_*` 环境变量
- MPI: 生成 SSH 密钥、创建 hostfile
- DeepSpeed: 配置 DeepSpeed launcher

### 扩展类型

- [x] **接口扩展**: 实现 `EnforceMLPolicyPlugin` 接口
- [ ] 插件扩展: 动态加载
- [ ] Webhook 扩展
- [ ] 配置扩展

### 接口定义

```go
// pkg/runtime/framework/interface.go:40-48
type EnforceMLPolicyPlugin interface {
    // Name 返回插件名称
    Name() string

    // EnforceMLPolicy 在 Info 上注入 ML 框架特定配置
    EnforceMLPolicy(
        info *runtime.Info,
        trainJob *trainer.TrainJob,
        runtimeSpec *trainer.TrainingRuntimeSpec,
    ) *runtime.Info
}
```

### 注册机制

```go
// pkg/runtime/framework/core/framework.go:69-90
func New(ctx context.Context, client client.Client, opts ...Option) (*Framework, error) {
    f := &Framework{}

    // 注册 MLPolicy 插件
    for name, factory := range options.enforceMLPolicyPlugins {
        f.enforceMLPolicyPlugins = append(f.enforceMLPolicyPlugins,
            factory(client))
    }

    return f, nil
}

// 默认插件注册
// pkg/runtime/framework/plugins/registry.go
var DefaultEnforceMLPolicyPlugins = map[string]EnforceMLPolicyPluginFactory{
    torch.Name:     torch.New,
    mpi.Name:       mpi.New,
    deepspeed.Name: deepspeed.New,
}
```

### 调用时机

```
TrainJob Reconcile:
    Get TrainJob
         │
         ▼
    Get RuntimeSpec
         │
         ▼
    ┌────────────────────────────────────────────┐
    │  RunEnforceMLPolicyPlugins                 │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
    │  │  Torch   │→│   MPI    │→│DeepSpeed │   │
    │  └──────────┘ └──────────┘ └──────────┘   │
    └────────────────────────────────────────────┘
         │
         ▼
    RunEnforceCustomizationPlugins
         │
         ▼
    BuildObjects (JobSet)
```

### 隔离与配额

| 隔离机制 | 配置 | 目的 |
|:---------|:-----|:-----|
| 接口契约 | 只能修改 Info | 防止插件越权 |
| 执行顺序 | 固定顺序 | 保证确定性 |
| 错误处理 | 返回 nil 跳过 | 单插件失败不影响其他 |

### 现有实现

| 实现 | 用途 | 源码位置 |
|:-----|:-----|:---------|
| Torch | PyTorch 分布式训练 | `pkg/runtime/framework/plugins/torch/` |
| MPI | MPI 分布式训练 | `pkg/runtime/framework/plugins/mpi/` |
| DeepSpeed | DeepSpeed 训练 | `pkg/runtime/framework/plugins/deepspeed/` |

---

## Extension Point 2: EnforceCustomization

### 设计意图

支持不同的调度策略和资源定制：
- Gang 调度: Volcano 或 Coscheduling
- PodGroup 配置
- 自定义 Pod 模板覆盖

### 接口定义

```go
// pkg/runtime/framework/interface.go:50-58
type EnforceCustomizationPlugin interface {
    Name() string

    EnforceCustomization(
        info *runtime.Info,
        trainJob *trainer.TrainJob,
        runtimeSpec *trainer.TrainingRuntimeSpec,
    ) *runtime.Info
}
```

### 现有实现

| 实现 | 用途 | 源码位置 |
|:-----|:-----|:---------|
| Coscheduling | scheduler-plugins Gang 调度 | `pkg/runtime/framework/plugins/coscheduling/` |
| Volcano | Volcano Gang 调度 | `pkg/runtime/framework/plugins/volcano/` |
| JobSet | JobSet 模板定制 | `pkg/runtime/framework/plugins/jobset/` |

---

## Extension Point 3: ComponentBuilder

### 设计意图

构建 TrainJob 所需的 Kubernetes 资源对象：
- JobSet: 核心工作负载
- Secret: MPI SSH 密钥
- ConfigMap: MPI hostfile

### 接口定义

```go
// pkg/runtime/framework/interface.go:60-72
type ComponentBuilderPlugin interface {
    Name() string

    // Build 构建 Kubernetes 对象
    Build(
        ctx context.Context,
        info *runtime.Info,
        trainJob *trainer.TrainJob,
        runtimeSpec *trainer.TrainingRuntimeSpec,
    ) ([]client.Object, error)
}
```

### 构建流程

```
RunComponentBuilderPlugins:
    ┌──────────────────────────────────────────────┐
    │  JobSet Plugin                               │
    │  ┌────────────────────────────────────────┐  │
    │  │ 1. 构建 JobSet spec                    │  │
    │  │ 2. 应用 PodSpecOverrides               │  │
    │  │ 3. 设置 OwnerReference                 │  │
    │  └────────────────────────────────────────┘  │
    │                    │                         │
    │                    ▼                         │
    │  ┌────────────────────────────────────────┐  │
    │  │ Output: JobSet ApplyConfiguration      │  │
    │  └────────────────────────────────────────┘  │
    └──────────────────────────────────────────────┘
         │
         │  (如果是 MPI 训练)
         ▼
    ┌──────────────────────────────────────────────┐
    │  MPI Plugin (Build 阶段)                     │
    │  ┌────────────────────────────────────────┐  │
    │  │ 1. 生成 SSH 密钥对                     │  │
    │  │ 2. 创建 Secret                         │  │
    │  │ 3. 创建 hostfile ConfigMap             │  │
    │  └────────────────────────────────────────┘  │
    │                    │                         │
    │                    ▼                         │
    │  ┌────────────────────────────────────────┐  │
    │  │ Output: Secret + ConfigMap             │  │
    │  └────────────────────────────────────────┘  │
    └──────────────────────────────────────────────┘
```

---

## Extension Point 4: Webhook (Validate)

### 设计意图

在资源创建/更新时进行验证：
- 检查保留环境变量
- 检查 PodTemplateOverrides 修改条件
- 自定义验证规则

### 接口定义

```go
// pkg/runtime/framework/interface.go:74-82
type WatchExtensionPlugin interface {
    Name() string

    // Validate 验证 TrainJob
    Validate(oldObj, newObj *trainer.TrainJob) (admission.Warnings, field.ErrorList)
}
```

### 调用时机

```
Webhook (Validating):
    Create/Update TrainJob
         │
         ▼
    ┌──────────────────────────────────────────────┐
    │  Framework.RunWatchExtensionPlugins          │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐     │
    │  │  Torch   │→│  JobSet  │→│   ...    │     │
    │  │ Validate │ │ Validate │ │          │     │
    │  └──────────┘ └──────────┘ └──────────┘     │
    │                    │                         │
    │                    ▼                         │
    │       合并所有 Warnings 和 Errors            │
    └──────────────────────────────────────────────┘
         │
         ▼
    允许/拒绝请求
```

---

## Extension Point 5: Runtime

### 设计意图

支持不同作用域的 Runtime：
- TrainingRuntime: 命名空间级
- ClusterTrainingRuntime: 集群级

### 接口定义

```go
// pkg/runtime/interface.go:26-38
type Runtime interface {
    // GetRuntimeSpec 获取 Runtime 规格
    GetRuntimeSpec(
        ctx context.Context,
        trainJob *trainer.TrainJob,
    ) (*trainer.TrainingRuntimeSpec, error)

    // BuildObjects 构建 Kubernetes 对象
    BuildObjects(
        ctx context.Context,
        info *Info,
        trainJob *trainer.TrainJob,
        runtimeSpec *trainer.TrainingRuntimeSpec,
    ) ([]client.Object, error)
}
```

### 注册机制

```go
// pkg/runtime/core/registry.go
func NewRuntimeRegistry(client client.Client, fwk *framework.Framework) RuntimeRegistry {
    return map[string]runtime.Runtime{
        // 命名空间级 Runtime
        runtimeRefKey(trainer.TrainingRuntimeKind): NewTrainingRuntime(client, fwk),

        // 集群级 Runtime
        runtimeRefKey(trainer.ClusterTrainingRuntimeKind): NewClusterTrainingRuntime(client, fwk),
    }
}
```

---

## 扩展点治理

### 生命周期

```
Alpha (实验性) → Beta (功能完整) → Stable (生产可用)
     │                                    │
     └────── 可能被移除或大幅修改 ─────────┘
```

当前各扩展点状态：

| 扩展点 | 状态 | 说明 |
|:-------|:-----|:-----|
| EnforceMLPolicy | Beta | Torch/MPI 稳定，DeepSpeed 开发中 |
| EnforceCustomization | Beta | Coscheduling/Volcano 支持 |
| ComponentBuilder | Beta | JobSet 构建稳定 |
| Webhook | Stable | 验证机制成熟 |
| Runtime | Stable | 核心接口稳定 |

### 兼容性规则

- [x] 新增扩展点：向后兼容，新插件不影响现有功能
- [x] 修改扩展接口：需要版本化（v1alpha1 → v1beta1）
- [ ] 移除扩展点：提前 2 个版本标记 Deprecated

---

## 如何添加新插件

### 步骤 1: 实现接口

```go
// pkg/runtime/framework/plugins/myplugin/myplugin.go
package myplugin

const Name = "MyPlugin"

type MyPlugin struct {
    client client.Client
}

func New(client client.Client) framework.EnforceMLPolicyPlugin {
    return &MyPlugin{client: client}
}

func (p *MyPlugin) Name() string {
    return Name
}

func (p *MyPlugin) EnforceMLPolicy(
    info *runtime.Info,
    trainJob *trainer.TrainJob,
    runtimeSpec *trainer.TrainingRuntimeSpec,
) *runtime.Info {
    // 实现逻辑
    return info
}
```

### 步骤 2: 注册插件

```go
// pkg/runtime/framework/plugins/registry.go
var DefaultEnforceMLPolicyPlugins = map[string]EnforceMLPolicyPluginFactory{
    torch.Name:     torch.New,
    mpi.Name:       mpi.New,
    myplugin.Name:  myplugin.New,  // 新增
}
```

### 步骤 3: 添加测试

```go
// pkg/runtime/framework/plugins/myplugin/myplugin_test.go
func TestMyPluginEnforceMLPolicy(t *testing.T) {
    // 测试用例
}
```

---

## 反模式警示

| 反模式 | 描述 | 风险 |
|:-------|:-----|:-----|
| 插件相互依赖 | Plugin A 依赖 Plugin B 的输出 | 顺序敏感，难以测试 |
| 过度扩展 | 所有配置都做成插件 | 复杂度爆炸 |
| 接口过于宽泛 | Info 包含太多可修改字段 | 难以追踪变更 |

---

## 评估矩阵

| 扩展点 | 使用频率 | 实现数量 | 稳定性 | 文档完善度 |
|:-------|:---------|:---------|:-------|:-----------|
| EnforceMLPolicy | 高 | 3 | Beta | 完善 |
| EnforceCustomization | 中 | 3 | Beta | 完善 |
| ComponentBuilder | 高 | 2 | Beta | 完善 |
| Webhook | 中 | 2 | Stable | 待改进 |
| Runtime | 高 | 2 | Stable | 完善 |
