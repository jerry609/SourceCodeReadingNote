# Plugin Framework

## 职责

Plugin Framework 是 Trainer 的可扩展性核心，提供：

1. 插件注册和生命周期管理
2. 按类型分组的插件执行
3. 统一的插件接口定义
4. 插件间的数据共享 (通过 `runtime.Info`)

## 核心接口

```go
// pkg/runtime/framework/interface.go

// 基础插件接口
type Plugin interface {
    Name() string
}

// 自定义验证插件
type CustomValidationPlugin interface {
    Plugin
    Validate(ctx context.Context, info *runtime.Info, oldObj, newObj *trainer.TrainJob) (admission.Warnings, field.ErrorList)
}

// Watch 扩展插件
type WatchExtensionPlugin interface {
    Plugin
    ReconcilerBuilders() []runtime.ReconcilerBuilder
}

// PodGroup 策略执行插件
type EnforcePodGroupPolicyPlugin interface {
    Plugin
    EnforcePodGroupPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error
}

// ML 策略执行插件
type EnforceMLPolicyPlugin interface {
    Plugin
    EnforceMLPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error
}

// Pod 网络插件
type PodNetworkPlugin interface {
    Plugin
    IdentifyPodNetwork(info *runtime.Info, trainJob *trainer.TrainJob) error
}

// 组件构建插件
type ComponentBuilderPlugin interface {
    Plugin
    Build(ctx context.Context, info *runtime.Info, trainJob *trainer.TrainJob) ([]apiruntime.ApplyConfiguration, error)
}

// TrainJob 状态插件
type TrainJobStatusPlugin interface {
    Plugin
    Status(ctx context.Context, trainJob *trainer.TrainJob) (*trainer.TrainJobStatus, error)
}
```

## 插件注册表

```go
// pkg/runtime/framework/plugins/registry.go
type Registry map[string]func(ctx context.Context, client client.Client, indexer client.FieldIndexer) (framework.Plugin, error)

func NewRegistry() Registry {
    return Registry{
        coscheduling.Name: coscheduling.New,  // Gang 调度 (scheduler-plugins)
        volcano.Name:      volcano.New,        // Gang 调度 (Volcano)
        mpi.Name:          mpi.New,            // MPI 分布式训练
        plainml.Name:      plainml.New,        // 基础 ML 支持
        torch.Name:        torch.New,          // PyTorch 分布式训练
        jobset.Name:       jobset.New,         // JobSet 资源构建
    }
}
```

## 框架核心实现

```go
// pkg/runtime/framework/core/framework.go
type Framework struct {
    registry                     fwkplugins.Registry
    plugins                      map[string]framework.Plugin

    // 按类型分组的插件列表
    enforceMLPlugins             []framework.EnforceMLPolicyPlugin
    enforcePodGroupPolicyPlugins []framework.EnforcePodGroupPolicyPlugin
    customValidationPlugins      []framework.CustomValidationPlugin
    watchExtensionPlugins        []framework.WatchExtensionPlugin
    podNetworkPlugins            []framework.PodNetworkPlugin
    componentBuilderPlugins      []framework.ComponentBuilderPlugin
    trainJobStatusPlugin         framework.TrainJobStatusPlugin  // 只允许一个
}
```

### 框架初始化

```go
// pkg/runtime/framework/core/framework.go:49-91
func New(ctx context.Context, c client.Client, r fwkplugins.Registry, indexer client.FieldIndexer) (*Framework, error) {
    f := &Framework{registry: r}
    plugins := make(map[string]framework.Plugin, len(r))

    // 设置索引器
    if err := f.SetupRuntimeClassIndexer(ctx, indexer); err != nil {
        return nil, err
    }

    // 遍历注册表，初始化每个插件
    for name, factory := range r {
        plugin, err := factory(ctx, c, indexer)
        if err != nil {
            return nil, err
        }
        plugins[name] = plugin

        // 根据插件实现的接口，分组到对应列表
        if p, ok := plugin.(framework.EnforceMLPolicyPlugin); ok {
            f.enforceMLPlugins = append(f.enforceMLPlugins, p)
        }
        if p, ok := plugin.(framework.EnforcePodGroupPolicyPlugin); ok {
            f.enforcePodGroupPolicyPlugins = append(f.enforcePodGroupPolicyPlugins, p)
        }
        // ... 其他接口类型
        if p, ok := plugin.(framework.TrainJobStatusPlugin); ok {
            if f.trainJobStatusPlugin != nil {
                return nil, errorTooManyTrainJobStatusPlugin
            }
            f.trainJobStatusPlugin = p
        }
    }
    f.plugins = plugins
    return f, nil
}
```

### 插件执行方法

```go
// 运行 ML 策略插件
func (f *Framework) RunEnforceMLPolicyPlugins(info *runtime.Info, trainJob *trainer.TrainJob) error {
    for _, plugin := range f.enforceMLPlugins {
        if err := plugin.EnforceMLPolicy(info, trainJob); err != nil {
            return err
        }
    }
    return nil
}

// 运行组件构建插件
func (f *Framework) RunComponentBuilderPlugins(ctx context.Context, info *runtime.Info, trainJob *trainer.TrainJob) ([]apiruntime.ApplyConfiguration, error) {
    var objs []apiruntime.ApplyConfiguration
    for _, plugin := range f.componentBuilderPlugins {
        components, err := plugin.Build(ctx, info, trainJob)
        if err != nil {
            return nil, err
        }
        objs = append(objs, components...)
    }
    return objs, nil
}
```

## 插件类型说明

| 插件类型 | 职责 | 实现插件 |
|:---------|:-----|:---------|
| `EnforceMLPolicyPlugin` | 设置 ML 框架相关的环境变量、命令参数 | Torch, MPI |
| `EnforcePodGroupPolicyPlugin` | 设置 PodGroup 相关的标签和配置 | Coscheduling, Volcano |
| `PodNetworkPlugin` | 识别 Pod 端点，设置网络信息 | JobSet |
| `ComponentBuilderPlugin` | 构建 Kubernetes 资源 | JobSet, MPI |
| `TrainJobStatusPlugin` | 获取 TrainJob 状态 | JobSet |
| `WatchExtensionPlugin` | 注册额外的 Watch 事件 | JobSet, MPI |
| `CustomValidationPlugin` | 自定义验证逻辑 | Torch, MPI, JobSet |

## 插件执行顺序

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Plugin Execution Pipeline                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. EnforceMLPolicy Plugins (Torch, MPI)                               │
│     └── 设置分布式训练环境变量、节点数、进程数                            │
│                                                                         │
│  2. EnforcePodGroupPolicy Plugins (Coscheduling, Volcano)              │
│     └── 设置 Gang 调度相关标签                                          │
│                                                                         │
│  3. PodNetwork Plugins (JobSet)                                        │
│     └── 识别 Pod 端点，设置 Endpoints 迭代器                            │
│                                                                         │
│  4. ComponentBuilder Plugins (JobSet, MPI)                             │
│     └── 构建 JobSet, Secret, ConfigMap 等资源                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 设计模式

### 1. 接口隔离原则

每个插件只需实现其关心的接口，框架通过类型断言自动分类：

```go
// Torch 插件只实现 EnforceMLPolicyPlugin 和 CustomValidationPlugin
type Torch struct{}

var _ framework.EnforceMLPolicyPlugin = (*Torch)(nil)
var _ framework.CustomValidationPlugin = (*Torch)(nil)
```

### 2. 开闭原则

新增插件无需修改框架代码，只需：
1. 实现 `Plugin` 接口及所需的扩展接口
2. 在 `Registry` 中注册

### 3. 单一职责

每个插件专注于特定功能：
- `Torch`: PyTorch 分布式训练配置
- `MPI`: MPI hostfile 和 SSH 密钥生成
- `JobSet`: JobSet 资源构建和状态同步

## 插件间通信

插件通过 `runtime.Info` 对象共享数据：

```go
// Torch 插件修改 Info 中的环境变量
func (t *Torch) EnforceMLPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error {
    trainerContainer := info.FindContainerByPodSetAncestorContainerName(constants.AncestorTrainer, constants.Node)
    if trainerContainer != nil {
        apply.UpsertEnvVars(&trainerContainer.Env,
            *corev1ac.EnvVar().WithName("PET_NNODES").WithValue("4"),
        )
    }
    return nil
}

// JobSet 插件读取 Info 中的环境变量构建 JobSet
func (j *JobSet) Build(ctx context.Context, info *runtime.Info, trainJob *trainer.TrainJob) ([]apiruntime.ApplyConfiguration, error) {
    for psIdx, ps := range info.TemplateSpec.PodSets {
        for containerIdx, container := range ps.Containers {
            // 使用 Torch 插件设置的环境变量
            apply.UpsertEnvVars(&jobSetSpec.ReplicatedJobs[psIdx].Template.Spec.Template.Spec.Containers[containerIdx].Env,
                container.Env...,
            )
        }
    }
    // ...
}
```

## 与 Kubernetes Scheduler 插件框架的对比

| 特性 | Trainer Plugin Framework | Kubernetes Scheduler Framework |
|:-----|:-------------------------|:-------------------------------|
| 插件注册 | 静态 Registry | 动态配置文件 |
| 扩展点数量 | 7 个 | 10+ 个 |
| 插件通信 | `runtime.Info` 对象 | `CycleState` |
| 执行模式 | 顺序执行 | 顺序 + 并行 |
| 错误处理 | 快速失败 | 可配置过滤 |

Trainer 的插件框架借鉴了 Kubernetes Scheduler 的设计，但进行了简化，更适合资源生成场景。
