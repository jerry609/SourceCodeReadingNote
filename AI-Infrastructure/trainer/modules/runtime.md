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
    Scheduler Scheduler

    // 模板规格
    TemplateSpec TemplateSpec
}

type RuntimePolicy struct {
    *trainer.MLPolicy           // ML 策略 (Torch/MPI)
    *trainer.PodGroupPolicy     // PodGroup 策略 (Coscheduling/Volcano)
}

type TemplateSpec struct {
    apply any           // JobSet Apply Configuration
    PodSets []PodSet    // Pod 集合信息
}

type PodSet struct {
    Name       string
    Count      *int32
    Endpoints  iter.Seq[string]    // Pod 端点迭代器
    Volumes    []corev1ac.VolumeApplyConfiguration
    Containers []Container
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
// pkg/runtime/core/registry.go
func New(ctx context.Context, client client.Client, indexer client.FieldIndexer) (map[string]runtime.Runtime, error) {
    // 创建插件框架
    fwk, err := fwkcore.New(ctx, client, plugins.NewRegistry(), indexer)
    if err != nil {
        return nil, err
    }

    // 注册支持的 Runtime 类型
    runtimes := map[string]runtime.Runtime{
        // TrainingRuntime (命名空间级)
        runtime.RuntimeRefToRuntimeRegistryKey(trainer.RuntimeRef{
            APIGroup: ptr.To(trainer.GroupVersion.Group),
            Kind:     ptr.To(trainer.TrainingRuntimeKind),
        }): NewTrainingRuntime(client, fwk),

        // ClusterTrainingRuntime (集群级)
        runtime.RuntimeRefToRuntimeRegistryKey(trainer.RuntimeRef{
            APIGroup: ptr.To(trainer.GroupVersion.Group),
            Kind:     ptr.To(trainer.ClusterTrainingRuntimeKind),
        }): NewClusterTrainingRuntime(client, fwk),
    }
    return runtimes, nil
}
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

    // 2. 构建 RuntimeInfo
    info, err := r.RuntimeInfo(trainJob, &runtime.Spec.Template.Spec,
        runtime.Spec.MLPolicy, runtime.Spec.PodGroupPolicy)
    if err != nil {
        return nil, err
    }

    // 3. 运行 ML 策略插件 (Torch/MPI)
    if err := r.framework.RunEnforceMLPolicyPlugins(info, trainJob); err != nil {
        return nil, err
    }

    // 4. 运行 PodGroup 策略插件 (Coscheduling/Volcano)
    if err := r.framework.RunEnforcePodGroupPolicyPlugins(info, trainJob); err != nil {
        return nil, err
    }

    // 5. 运行 Pod 网络插件 (设置 Pod 端点)
    if err := r.framework.RunPodNetworkPlugins(info, trainJob); err != nil {
        return nil, err
    }

    // 6. 运行组件构建插件 (生成 JobSet, Secret, ConfigMap 等)
    return r.framework.RunComponentBuilderPlugins(ctx, info, trainJob)
}
```

### RuntimeInfo 构建

```go
// pkg/runtime/core/core.go
func (r *runtime) RuntimeInfo(
    trainJob *trainer.TrainJob,
    runtimeTemplateSpec any,
    mlPolicy *trainer.MLPolicy,
    podGroupPolicy *trainer.PodGroupPolicy,
) (*runtime.Info, error) {
    info := &runtime.Info{
        Labels:      make(map[string]string),
        Annotations: make(map[string]string),
        RuntimePolicy: runtime.RuntimePolicy{
            MLPolicy:       mlPolicy.DeepCopy(),
            PodGroupPolicy: podGroupPolicy.DeepCopy(),
        },
    }

    // 解析 JobSet 模板
    jobSetSpec, ok := runtimeTemplateSpec.(*jobsetv1alpha2ac.JobSetSpecApplyConfiguration)
    if !ok {
        return nil, fmt.Errorf("unsupported template spec type")
    }

    // 构建 PodSet 列表
    info.TemplateSpec.apply = jobSetSpec
    for _, rJob := range jobSetSpec.ReplicatedJobs {
        podSet := runtime.PodSet{
            Name:  *rJob.Name,
            Count: rJob.Template.Spec.Parallelism,
        }
        // 解析容器
        for _, c := range rJob.Template.Spec.Template.Spec.Containers {
            podSet.Containers = append(podSet.Containers, runtime.Container{
                Name: *c.Name,
                Env:  c.Env,
            })
        }
        info.TemplateSpec.PodSets = append(info.TemplateSpec.PodSets, podSet)
    }

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

各插件按顺序执行，每个插件修改 `Info` 对象：

```
EnforceMLPolicy (Torch) -> EnforceMLPolicy (MPI) ->
EnforcePodGroupPolicy (Coscheduling) -> PodNetwork (JobSet) ->
ComponentBuilder (JobSet) -> ComponentBuilder (MPI)
```

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
