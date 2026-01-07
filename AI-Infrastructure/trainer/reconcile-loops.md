# 闭环设计 (Reconcile Loops)

> 能演进的系统是控制回路：观察 → 计算 → 纠偏 → 再观察。

Trainer 的“闭环”主要靠 controller-runtime 的 Watch + Reconcile 实现，差异点在于：
- 期望状态由 TrainJob + (Cluster)TrainingRuntime + 插件共同决定
- 纠偏动作使用 **Server-Side Apply**（SSA）下发一组对象（JobSet/PodGroup/Secret/ConfigMap…）
- 状态从 JobSet 聚合回 TrainJob（由 JobSet 插件实现）

## 核心闭环清单

| 闭环名称 | 触发源 | 纠偏对象 | 幂等性手段 | 源码位置 |
|:--|:--|:--|:--|:--|
| TrainJob Reconcile | Watch TrainJob + 扩展 Watch | Apply 子资源 | SSA + FieldOwner | `pkg/controller/trainjob_controller.go` |
| Status 回写 | JobSet 状态变化触发 Reconcile | TrainJob.Status | MergeFrom Patch | `pkg/controller/trainjob_controller.go` + JobSet 插件 |
| Runtime 防误删 | TrainJob 变化触发 Runtime reconcile | Runtime Finalizer | Finalizer | `pkg/controller/trainingruntime_controller.go`, `pkg/controller/clustertrainingruntime_controller.go` |

外部闭环（不属于 Trainer，但强相关）：
- JobSet Controller：根据 JobSet spec 管理 Jobs/Pods 并更新 JobSet.Status
- Gang 调度（可选）：scheduler-plugins / Volcano 根据 PodGroup + labels 做调度决策

---

## Reconcile Loop 1：TrainJob Controller

### 闭环模型

```
Watch Events
  (TrainJob / JobSet / Secret / ConfigMap / PodGroup / ...)
      │
      ▼
Reconcile(trainJob)
  ├─ 读取 TrainJob
  ├─ 选择 Runtime（按 RuntimeRef Group/Kind）
  ├─ 计算期望对象：runtime.NewObjects(...)
  ├─ 纠偏：client.Apply(SSA) 子资源
  ├─ 计算状态：runtime.TrainJobStatus(...)
  └─ 回写：client.Status().Patch(...)
```

### 触发条件（Observe）

基础触发：
- TrainJob create/update/delete（controller 自己 watch）

扩展触发：由插件通过 `WatchExtensionPlugin` 注入（见 `pkg/runtime/framework/interface.go`）
- JobSet：watch JobSet，owner 为 TrainJob 时 enqueue（JobSet 插件）
- MPI：watch Secret/ConfigMap（owner 为 TrainJob 时 enqueue，MPI 插件）
- Gang 调度相关：watch PodGroup / RuntimeClass / LimitRange（CoScheduling/Volcano 插件）

这类触发让系统更“水平触发（level-triggered）”：任何相关资源变化都会触发一次重新计算与收敛。

### 期望状态计算（Compute）

`runtime.NewObjects(ctx, trainJob)` 的内部结构可概括为两步：
1) 构建 `runtime.Info`（`RuntimeInfo(...)`）
   - 从 (Cluster)TrainingRuntime 的 JobSet 模板抽取 PodSets（作为插件操作的抽象对象）
   - 合并 TrainJob 的 labels/annotations 与 podTemplateOverrides
   - 执行插件阶段：MLPolicy → PodGroupPolicy → PodNetwork
2) 执行 `ComponentBuilder` 插件：输出最终要 SSA Apply 的对象列表

阶段顺序固定，但同一阶段内的插件顺序来自 registry（Go map 无序），因此插件应避免顺序依赖（一般通过 nil-check/ancestor-check 保证互斥或可交换）。

### 纠偏动作（Act）

TrainJob controller 的纠偏非常直接：对 runtime 输出的每个对象做 SSA Apply：

```go
// pkg/controller/trainjob_controller.go:reconcileObjects
objects, err := runtime.NewObjects(ctx, trainJob)
for _, object := range objects {
    err = r.client.Apply(ctx, object, client.FieldOwner("trainer"), client.ForceOwnership)
}
```

特性：
- 幂等：重复 Apply 收敛到同一结果
- 冲突处理：通过 FieldOwner + ForceOwnership 解决字段所有权冲突（代价是可能覆盖其他 owner 的字段）

---

## Reconcile Loop 2：Status 回写

### 状态来源

Trainer 本身不直接管理 Pod/Job 状态，主要依赖 JobSet：
- JobSet controller 更新 `JobSet.Status.Conditions` 与 `ReplicatedJobsStatus`
- JobSet 插件实现 `TrainJobStatusPlugin`，在 `Status(ctx, trainJob)` 中读取 JobSet 并映射到 TrainJob

典型映射：
- `JobSetCompleted` → `TrainJobComplete`
- `JobSetFailed` → `TrainJobFailed`

### 回写机制

TrainJob controller 在每次 Reconcile 尾部：
- `runtime.TrainJobStatus(...)`
- 如果 `trainJob.Status` 发生变化：`client.Status().Patch(..., MergeFrom(prev))`

这里暂未使用 SSA（代码里有 TODO：等待 controller-runtime 对 subresource SSA 的支持）。

---

## Reconcile Loop 3：Runtime 防误删（Finalizer）

TrainingRuntime / ClusterTrainingRuntime controller 的主要工作是维护 `ResourceInUseFinalizer`：
- 若存在 TrainJob 引用该 runtime：为 runtime 加 finalizer（阻止被删除）
- runtime 被删除（DeletionTimestamp 非空）且已无引用：移除 finalizer 允许删除

为了让 finalizer 更快收敛，TrainJob controller 会在 create/update/delete 事件后通过 watcher 通知 Runtime controller（见 `pkg/controller/setup.go` 里 `WithWatchers(...)` 的注入）。

---

## 失败处理与退避

- Reconcile 返回 error：交由 controller-runtime workqueue 处理重试与退避
- Runtime 不支持：设置 `TrainJobFailed` condition（`TrainJobRuntimeNotSupportedReason`）
- 子资源构建失败：记录 Event（`TrainJobResourcesCreationFailed`），并等待下一次重试触发

