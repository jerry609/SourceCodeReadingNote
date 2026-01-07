# Runtime 模块

## 职责

Runtime 模块是 Trainer 的核心抽象层，负责：

1. 解析 TrainingRuntime/ClusterTrainingRuntime 模板
2. 合并 TrainJob 配置与运行时模板
3. 协调各插件生成 Kubernetes 资源
4. 提供 TrainJob 状态查询

## 核心接口

```go
// pkg/runtime/interface.go:34-40
type Runtime interface {
    // 根据 TrainJob 生成 Kubernetes 资源配置
    NewObjects(ctx context.Context, trainJob *trainer.TrainJob) ([]runtime.ApplyConfiguration, error)

    // 获取运行时信息（用于插件）
    RuntimeInfo(trainJob *trainer.TrainJob, runtimeTemplateSpec any,
        mlPolicy *trainer.MLPolicy, podGroupPolicy *trainer.PodGroupPolicy) (*Info, error)

    // 获取 TrainJob 状态
    TrainJobStatus(ctx context.Context, trainJob *trainer.TrainJob) (*trainer.TrainJobStatus, error)

    // 注册事件处理器
    EventHandlerRegistrars() []ReconcilerBuilder

    // 验证资源
    ValidateObjects(ctx context.Context, old, new *trainer.TrainJob) (admission.Warnings, field.ErrorList)
}
```

## 核心数据结构

### Info - 运行时信息

```go
// pkg/runtime/runtime.go
type Info struct {
    // 标签和注解
    Labels      map[string]string
    Annotations map[string]string

    // 运行时策略
    RuntimePolicy RuntimePolicy

    // 调度器配置
    Scheduler *Scheduler

    // 模板规格
    TemplateSpec TemplateSpec
}

type RuntimePolicy struct {
    MLPolicySource *trainer.MLPolicySource
    PodGroupPolicy *trainer.PodGroupPolicy
}

type TemplateSpec struct {
    ObjApply any        // (Cluster)TrainingRuntime.Template 对应的 ApplyConfiguration
    PodSets   []PodSet  // 从 ObjApply 抽取出的 PodSpec 抽象
}

type PodSet struct {
    Name            string
    Ancestor        *string
    Count           *int32
    InitContainers  []Container
    Containers      []Container
    Volumes         []corev1ac.VolumeApplyConfiguration
    Endpoints       iter.Seq[string] // Pod 端点迭代器（由 PodNetwork 插件填充）
    SinglePodRequests corev1.ResourceList // 资源请求（用于 PodGroup 聚合）
}

type Container struct {
    Name         string
    Env          []corev1ac.EnvVarApplyConfiguration
    Ports        []corev1ac.ContainerPortApplyConfiguration
    VolumeMounts []corev1ac.VolumeMountApplyConfiguration
}

type Scheduler struct {
    PodLabels      map[string]string
    PodAnnotations map[string]string
}
```

## 架构设计

```
┌───────────────────────────────────────────────────────────────────────┐
│                         Runtime Layer                                  │
├───────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                    TrainingRuntime Core                         │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │  NewObjects(trainJob)                                     │  │   │
│  │  │    1. Get TrainingRuntime by runtimeRef                   │  │   │
│  │  │    2. Build RuntimeInfo from template                     │  │   │
│  │  │    3. Run EnforceMLPolicy plugins                         │  │   │
│  │  │    4. Run EnforcePodGroupPolicy plugins                   │  │   │
│  │  │    5. Run PodNetwork plugins                              │  │   │
│  │  │    6. Run ComponentBuilder plugins                        │  │   │
│  │  │    7. Return [JobSet, PodGroup, Secret, ConfigMap, ...]   │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                 │                                      │
│                                 ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │                      Plugin Framework                           │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐   │   │
│  │  │ Torch  │ │  MPI   │ │ JobSet │ │CoSched │ │  Volcano   │   │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────────┘   │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
└───────────────────────────────────────────────────────────────────────┘
```

## 交互关系

**调用方**:
- TrainJob Controller
- Webhooks (验证)

**依赖方**:
- Plugin Framework
- Kubernetes API (获取 TrainingRuntime)

## 源码走读记录

### Runtime 注册表初始化

```go
// pkg/runtime/core/core.go:New（节选）
registry := NewRuntimeRegistry()
runtimes := make(map[string]runtime.Runtime, len(registry))
for name, registrar := range registry {
    // 处理依赖（ClusterTrainingRuntime 依赖 TrainingRuntime 先初始化）
    for _, dep := range registrar.dependencies { /* ... */ }
    if _, ok := runtimes[name]; !ok {
        r, err := registrar.factory(ctx, client, indexer)
        if err != nil { return nil, err }
        runtimes[name] = r
    }
}
return runtimes, nil
```

### NewObjects 核心流程

```go
// pkg/runtime/core/trainingruntime.go (简化版)
func (r *TrainingRuntime) NewObjects(ctx context.Context, trainJob *trainer.TrainJob) ([]apiruntime.ApplyConfiguration, error) {
    // 1. 获取 TrainingRuntime
    var runtime trainer.TrainingRuntime
    if err := r.client.Get(ctx, client.ObjectKey{
        Name:      trainJob.Spec.RuntimeRef.Name,
        Namespace: trainJob.Namespace,
    }, &runtime); err != nil {
        return nil, err
    }

    // 2. 构建 RuntimeInfo（内部会执行 MLPolicy/PodGroupPolicy/PodNetwork 三类插件）
    info, err := r.RuntimeInfo(trainJob, runtime.Spec.Template,
        runtime.Spec.MLPolicy, runtime.Spec.PodGroupPolicy)
    if err != nil {
        return nil, err
    }

    // 3. 运行组件构建插件 (生成 JobSet, PodGroup, Secret, ConfigMap 等)
    return r.framework.RunComponentBuilderPlugins(ctx, info, trainJob)
}
```

### RuntimeInfo 构建

```go
// pkg/runtime/core/trainingruntime.go:RuntimeInfo（节选）
func (r *TrainingRuntime) RuntimeInfo(
    trainJob *trainer.TrainJob,
    runtimeTemplateSpec any,
    mlPolicy *trainer.MLPolicy,
    podGroupPolicy *trainer.PodGroupPolicy,
) (*runtime.Info, error) {
    jobSetTemplateSpec, ok := runtimeTemplateSpec.(trainer.JobSetTemplateSpec)
    if !ok { return nil, fmt.Errorf("unsupported runtimeTemplateSpec") }

    info, err := r.newRuntimeInfo(trainJob, jobSetTemplateSpec, mlPolicy, podGroupPolicy)
    if err != nil { return nil, err }

    // 固定阶段的插件管线
    if err = r.framework.RunEnforceMLPolicyPlugins(info, trainJob); err != nil { return nil, err }
    if err = r.framework.RunEnforcePodGroupPolicyPlugins(info, trainJob); err != nil { return nil, err }
    if err = r.framework.RunPodNetworkPlugins(info, trainJob); err != nil { return nil, err }
    return info, nil
}
```

### TrainJob 状态获取

```go
// pkg/runtime/core/trainingruntime.go
func (r *TrainingRuntime) TrainJobStatus(ctx context.Context, trainJob *trainer.TrainJob) (*trainer.TrainJobStatus, error) {
    // 委托给 TrainJobStatus 插件 (通常是 JobSet 插件)
    return r.framework.RunTrainJobStatusPlugin(ctx, trainJob)
}
```

## 关键设计模式

### 1. 模板 + 覆盖模式

TrainingRuntime 定义基础模板，TrainJob 提供覆盖配置：

```yaml
# TrainingRuntime 模板
spec:
  mlPolicy:
    numNodes: 1
    torch:
      numProcPerNode: "auto"

# TrainJob 覆盖
spec:
  trainer:
    numNodes: 4  # 覆盖节点数
    resourcesPerNode:
      limits:
        nvidia.com/gpu: 8
```

### 2. 插件链模式

各插件按“阶段”顺序执行，每个插件修改 `Info` 对象：

```
EnforceMLPolicy (PlainML/Torch/MPI) ->
EnforcePodGroupPolicy (CoScheduling/Volcano) ->
PodNetwork (JobSet) ->
ComponentBuilder (JobSet/MPI/CoScheduling/Volcano)
```

备注：同一阶段内的插件执行顺序来自 registry 的遍历（Go map 无序），因此插件应避免顺序依赖，通常通过 nil-check/ancestor-check 做互斥与隔离。

### 3. Apply Configuration 模式

使用 Kubernetes client-go 的 Apply Configuration 进行声明式资源管理：

```go
// 而非传统的 Create/Update，使用 SSA
jobSet := jobsetv1alpha2ac.JobSet(name, namespace).
    WithSpec(spec).
    WithOwnerReferences(ownerRef)

client.Apply(ctx, jobSet, client.FieldOwner("trainer"))
```

## 辅助函数

```go
// pkg/runtime/runtime.go

// 根据 Ancestor 标签查找 PodSet
func (info *Info) FindPodSetByAncestor(ancestor string) *PodSet

// 根据名称查找 PodSet
func (info *Info) FindPodSetByName(name string) *PodSet

// 根据 PodSet Ancestor 和容器名查找容器
func (info *Info) FindContainerByPodSetAncestorContainerName(ancestor, containerName string) *Container

// 从运行时模板提取每节点资源配置
func ExtractResourcePerNodeFromRuntime(info *Info) *corev1.ResourceRequirements

// 获取每节点 GPU 数量
func GetNumGPUPerNode(res *corev1.ResourceRequirements) int
```
