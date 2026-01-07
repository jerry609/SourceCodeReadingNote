# 扩展点设计 (Extension Points)

> 把变化安置在“插件边缘”，把稳定留在“内核中心”。

Trainer 的扩展点主要围绕两件事：
1) **把 (Cluster)TrainingRuntime 的模板变成可执行的 K8s 资源**（JobSet/PodGroup/Secret/ConfigMap…）
2) **把 ML/调度的差异收敛在插件里**（Torch/MPI、coscheduling/volcano 等）

## 设计原则

1. **识别变化**：ML 策略（Torch/MPI/无策略）、Gang 调度策略（Coscheduling/Volcano/无）
2. **封装变化**：把变化点做成可插拔的 Plugin，核心只关心“执行哪个阶段”
3. **保护核心**：Controller + Runtime 接口尽量稳定；插件通过 `runtime.Info` 交换数据

---

## 核心调用链（把扩展点放进时序里）

```
TrainJob Controller
  └─ runtime.NewObjects()
       ├─ runtime.RuntimeInfo()
       │    ├─ EnforceMLPolicy plugins
       │    ├─ EnforcePodGroupPolicy plugins
       │    └─ PodNetwork plugins
       └─ ComponentBuilder plugins   -> []ApplyConfiguration

TrainJob Webhook
  └─ runtime.ValidateObjects()
       └─ CustomValidation plugins   -> warnings + field.ErrorList

TrainJob Status
  └─ runtime.TrainJobStatus()
       └─ TrainJobStatus plugin      -> TrainJobStatus

Watches 扩展
  └─ runtime.EventHandlerRegistrars()
       └─ WatchExtension plugins     -> []ReconcilerBuilder
```

---

## 扩展点清单（以接口为准）

接口定义在：`pkg/runtime/framework/interface.go`，运行时接口在：`pkg/runtime/interface.go`。

| 扩展点（接口） | 解决的问题 | 典型实现 |
|:--|:--|:--|
| `EnforceMLPolicyPlugin` | 注入 ML 运行所需的 env/volume/count 等 | PlainML/Torch/MPI |
| `EnforcePodGroupPolicyPlugin` | 注入 Gang 调度所需的 Pod labels 等 | CoScheduling/Volcano |
| `PodNetworkPlugin` | 识别 Pod endpoints（用于 rendezvous/hostfile 等） | JobSet |
| `ComponentBuilderPlugin` | 构建最终要 Apply 的对象 | JobSet/MPI/CoScheduling/Volcano |
| `TrainJobStatusPlugin` | 从底层工作负载聚合 TrainJob 状态 | JobSet |
| `CustomValidationPlugin` | admission 阶段的自定义校验 | Torch/MPI/JobSet |
| `WatchExtensionPlugin` | 给 TrainJob controller 追加 watches | JobSet/MPI/CoScheduling/Volcano |

---

## Extension Point 1：`EnforceMLPolicyPlugin`

### 设计意图

把“如何跑起来”的差异收敛在一处：
- Torch：写入 `PET_*` 分布式 env；TorchTune 会补齐/校验命令参数
- MPI：调整 node 数、写入 slots、挂载 SSH/hostfile、注入 OpenMPI env
- PlainML：当 runtime 未配置 Torch/MPI 时，做最基础的 `numNodes/env` 透传

### 接口（精简版）

```go
type EnforceMLPolicyPlugin interface {
    Plugin
    EnforceMLPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error
}
```

### 调用位置

`pkg/runtime/core/trainingruntime.go:RuntimeInfo`（先构建 `runtime.Info`，再执行该阶段插件）。

---

## Extension Point 2：`EnforcePodGroupPolicyPlugin`

### 设计意图

Gang 调度器（scheduler-plugins / Volcano）的差异体现在：
- Pod 上需要哪些 labels/annotations
- 是否需要额外创建 PodGroup 资源

“注入 Pod labels”属于该阶段，真正创建 PodGroup 属于 `ComponentBuilderPlugin`（同一个插件可同时实现两者）。

### 接口（精简版）

```go
type EnforcePodGroupPolicyPlugin interface {
    Plugin
    EnforcePodGroupPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error
}
```

---

## Extension Point 3：`PodNetworkPlugin`

### 设计意图

统一生成“训练通信需要的 endpoints”：
- Torch：master addr（默认 `...-node-0-0.<subdomain>`）
- MPI：hostfile（每行一个 endpoint）

### 接口（精简版）

```go
type PodNetworkPlugin interface {
    Plugin
    IdentifyPodNetwork(info *runtime.Info, trainJob *trainer.TrainJob) error
}
```

---

## Extension Point 4：`ComponentBuilderPlugin`

### 设计意图

把 `runtime.Info`（抽象/中间态）变成最终要 Apply 的对象集合：
- JobSet（核心 workload）
- PodGroup（可选）
- MPI Secret / ConfigMap（可选）

### 接口（精简版）

```go
type ComponentBuilderPlugin interface {
    Plugin
    Build(ctx context.Context, info *runtime.Info, trainJob *trainer.TrainJob) ([]apiruntime.ApplyConfiguration, error)
}
```

---

## Extension Point 5：`CustomValidationPlugin`（Webhook 侧）

### 设计意图

把“用户能不能这样配”提前在 admission 阶段阻断：
- 运行中不可修改的字段（如 `podTemplateOverrides` 的限制）
- 运行时保留 env（如 Torch 的 `PET_*`）
- MPI/TorchTune 的参数约束

### 接口（精简版）

```go
type CustomValidationPlugin interface {
    Plugin
    Validate(ctx context.Context, info *runtime.Info, oldObj, newObj *trainer.TrainJob) (admission.Warnings, field.ErrorList)
}
```

---

## Extension Point 6：`WatchExtensionPlugin`

### 设计意图

TrainJob controller 默认只 watch TrainJob；但很多收敛点来自“派生资源变化”：
- JobSet 状态变化 -> 触发 TrainJob reconcile 更新 status
- MPI Secret/ConfigMap 变化 -> 触发 TrainJob reconcile
- PodGroup/RuntimeClass/LimitRange 变化（gang 调度相关）-> 触发（通常只对 suspended TrainJob 有意义）

### 接口（精简版）

```go
type WatchExtensionPlugin interface {
    Plugin
    ReconcilerBuilders() []runtime.ReconcilerBuilder
}
```

---

## 顺序、隔离与“不要写顺序依赖”

- 执行顺序按“阶段”固定：MLPolicy → PodGroupPolicy → PodNetwork → Build
- 同一阶段内的插件顺序来自 registry 的遍历（Go map 无序），**不应依赖先后**
- 常用的顺序隔离手段：
  - 配置守卫：`if info.RuntimePolicy.MLPolicySource.MPI == nil { return nil }`
  - ancestor 守卫：只修改特定 PodSet/Container

---

## 如何添加新插件（最短路径）

1) 实现所需接口（可实现多个）：

```go
package myplugin

const Name = "MyPlugin"

type MyPlugin struct{}

func New(context.Context, client.Client, client.FieldIndexer) (framework.Plugin, error) {
    return &MyPlugin{}, nil
}
func (p *MyPlugin) Name() string { return Name }
```

2) 在 `pkg/runtime/framework/plugins/registry.go:NewRegistry` 注册：
- `myplugin.Name: myplugin.New`

3) 加测试（优先放在插件目录下的 `_test.go`，对 `runtime.Info` 的变更与输出对象做断言）。

---

## 反模式警示

| 反模式 | 风险 |
|:--|:--|
| 插件 A 依赖插件 B 的执行顺序 | registry 无序 → 结果不确定，难以复现/测试 |
| 在插件里修改 TrainJob spec | 容易引入副作用；除非非常明确（例如 TorchTune 的命令补齐） |
| 让插件直接操作 client 写资源 | 破坏 SSA/幂等，难以追踪 ownership 与变更来源 |
