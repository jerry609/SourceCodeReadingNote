# TrainJob 创建/更新流程 (Reconcile)

本文从“请求进入集群”到“底层资源被下发/状态回写”，串起 TrainJob 的核心链路。

## 入口与关键文件

- Validating Webhook：`pkg/webhooks/trainjob_webhook.go`
- TrainJob Controller：`pkg/controller/trainjob_controller.go`
- Runtime：`pkg/runtime/interface.go`
- TrainingRuntime 实现：`pkg/runtime/core/trainingruntime.go`
- ClusterTrainingRuntime 实现：`pkg/runtime/core/clustertrainingruntime.go`
- 插件框架：`pkg/runtime/framework/core/framework.go`

## 总览时序（含 Webhook）

```
kubectl apply TrainJob
  ├─ admission: TrainJobWebhook.ValidateCreate/Update
  │    ├─ 选择 runtime 实现（按 RuntimeRef 的 Group/Kind）
  │    └─ runtime.ValidateObjects(...) -> Framework.RunCustomValidationPlugins(...)
  └─ controller: TrainJobReconciler.Reconcile
       ├─ 选择 runtime 实现（按 RuntimeRef 的 Group/Kind）
       ├─ runtime.NewObjects(...) -> (RuntimeInfo + ComponentBuilder plugins)
       ├─ SSA Apply 每个对象（JobSet/PodGroup/Secret/ConfigMap...）
       └─ runtime.TrainJobStatus(...) -> Patch TrainJob.Status
```

## 1) Webhook：创建/更新前的验证

TrainJob webhook 自身不实现业务校验，而是把校验“委托给 runtime”：
- `pkg/webhooks/trainjob_webhook.go`
  - `ValidateCreate/ValidateUpdate`
    - 通过 `runtime.RuntimeRefToRuntimeRegistryKey(trainJob.Spec.RuntimeRef)` 找到 runtime
    - `runtime.ValidateObjects(ctx, old, new)` 返回 `Warnings + field.ErrorList`

runtime 的验证实现（核心逻辑在 `TrainingRuntime.ValidateObjects` / `ClusterTrainingRuntime.ValidateObjects`）：
- 检查被引用的 (Cluster)TrainingRuntime 是否存在
- 构建一份 `runtime.Info`（忽略 runtime 本身无效的情况，假设 runtime 模板已是“平台侧正确配置”）
- 执行 `Framework.RunCustomValidationPlugins(...)`
  - 典型校验来源：
    - Torch：保留 env（`PET_*`）不允许用户覆盖、`numProcPerNode` 的取值约束、TorchTune 的约束
    - JobSet：`podTemplateOverrides` 可变性、initializer 配置与 runtime 模板是否匹配等
    - MPI：MPI 下 `numProcPerNode` 必须是 int、`runLauncherAsNode` 的 PodSet 约束等

## 2) Controller：Reconcile 主流程

核心路径：`pkg/controller/trainjob_controller.go:Reconcile`

### 2.1 获取对象与恢复语义

- `client.Get` 获取 TrainJob
- `removeFailedCondition(&trainJob)`
  - 目的：允许用户通过修改 spec 把 TrainJob 从 Failed 状态“拉回来”（即：Failed 不应成为永久状态）

### 2.2 选择 Runtime

runtime 的选择由 `RuntimeRef` 的 `apiGroup/kind` 决定：
- key：`schema.GroupKind{Group: RuntimeRef.APIGroup, Kind: RuntimeRef.Kind}.String()`
- registry 初始化来自启动阶段 `runtimecore.New(...)`（见 `flows/startup.md`）

如果 runtime 不支持：
- `setFailedCondition(..., TrainJobRuntimeNotSupportedReason)`

### 2.3 生成并下发子资源（SSA）

`reconcileObjects(ctx, runtime, &trainJob)`：
- `runtime.NewObjects(ctx, trainJob)` 返回一组 `ApplyConfiguration`
- 对每个对象执行 `client.Apply(..., FieldOwner("trainer"), ForceOwnership)`

其中 `runtime.NewObjects`（以 namespaced `TrainingRuntime` 为例）会做两层工作：
1) 构建 `runtime.Info`（`RuntimeInfo(...)`）
   - 合并 labels/annotations（runtime 模板 + `.spec.labels/.spec.annotations`）
   - 合并 `podTemplateOverrides`（对 PodTemplate 做 strategic merge patch）
   - 把 runtime 模板的 JobSet spec 转成 ApplyConfiguration，并抽取为 PodSets（后续插件操作的“抽象工作台”）
   - 执行固定阶段的插件管线：
     - `RunEnforceMLPolicyPlugins`
     - `RunEnforcePodGroupPolicyPlugins`
     - `RunPodNetworkPlugins`
2) 执行资源构建插件（`RunComponentBuilderPlugins`）
   - 典型输出：JobSet、PodGroup、MPI Secret/ConfigMap 等

注意：同一阶段内插件的顺序来自 registry 的遍历（Go map 无序），因此插件应避免顺序依赖；通常通过“配置守卫”（nil-check/ancestor-check）保证互斥或可交换。

### 2.4 条件与状态回写

- `setSuspendedCondition(&trainJob)`：把 `.spec.suspend` 反映到 `status.conditions`
- `setTrainJobStatus(ctx, runtime, &trainJob)`：
  - `runtime.TrainJobStatus(ctx, trainJob)` 委托给 `TrainJobStatusPlugin`（当前主要由 JobSet 插件实现）
  - 若 `status` 变化，`client.Status().Patch(..., MergeFrom(prev))`
    - 这里暂未使用 SSA（代码里有 TODO，等待 controller-runtime 对 subresource SSA 的支持）

