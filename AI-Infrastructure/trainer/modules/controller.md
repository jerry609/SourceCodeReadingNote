# Controller 模块

## 职责

Controller 模块负责协调 TrainJob、TrainingRuntime 和 ClusterTrainingRuntime 资源的生命周期。核心职责包括：

1. 监听 TrainJob 资源变化
2. 根据 RuntimeRef 查找对应的 TrainingRuntime
3. 调用 Runtime 层生成 Kubernetes 资源
4. 同步 TrainJob 状态

## 核心接口

```go
// pkg/controller/trainjob_controller.go:53-59
type TrainJobReconciler struct {
    log      logr.Logger
    client   client.Client
    recorder record.EventRecorder
    runtimes map[string]jobruntimes.Runtime  // 运行时注册表
    watchers iter.Seq[TrainJobWatcher]       // 事件观察者
}
```

## 交互关系

```
                    ┌─────────────────┐
                    │   Kubernetes    │
                    │   API Server    │
                    └────────┬────────┘
                             │ Watch
                             ▼
┌────────────────────────────────────────────────────────────┐
│                   TrainJob Controller                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Reconcile()                        │  │
│  │  1. Get TrainJob                                      │  │
│  │  2. Lookup Runtime by runtimeRef                      │  │
│  │  3. runtime.NewObjects() -> [JobSet, Secret, ...]    │  │
│  │  4. client.Apply() for each object                    │  │
│  │  5. Update TrainJob status                            │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Runtime      │
                    │  (Plugins)      │
                    └─────────────────┘
```

**调用方**:
- Kubernetes Controller Manager (通过 controller-runtime)

**依赖方**:
- Runtime 接口 (`pkg/runtime/interface.go`)
- Kubernetes API (通过 controller-runtime client)

## 源码走读记录

### Reconcile 主流程

```go
// pkg/controller/trainjob_controller.go:96-145
func (r *TrainJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取 TrainJob
    var trainJob trainer.TrainJob
    if err := r.client.Get(ctx, req.NamespacedName, &trainJob); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 清除之前可能设置的 Failed 条件
    removeFailedCondition(&trainJob)

    // 3. 根据 runtimeRef 查找对应的 Runtime
    runtimeRefGK := jobruntimes.RuntimeRefToRuntimeRegistryKey(trainJob.Spec.RuntimeRef)
    runtime, ok := r.runtimes[runtimeRefGK]
    if !ok {
        // 不支持的 runtime，设置 Failed 条件
        setFailedCondition(&trainJob, "unsupported runtime", trainer.TrainJobRuntimeNotSupportedReason)
    } else {
        // 4. 调用 Runtime 创建/更新资源
        err = r.reconcileObjects(ctx, runtime, &trainJob)
    }

    // 5. 设置 Suspended 条件
    setSuspendedCondition(&trainJob)

    // 6. 更新 TrainJob 状态
    if statusErr := setTrainJobStatus(ctx, runtime, &trainJob); statusErr != nil {
        err = errors.Join(err, statusErr)
    }

    // 7. Patch 状态
    if !equality.Semantic.DeepEqual(&trainJob.Status, prevTrainJob.Status) {
        return ctrl.Result{}, errors.Join(err, r.client.Status().Patch(ctx, &trainJob, client.MergeFrom(prevTrainJob)))
    }
    return ctrl.Result{}, err
}
```

### reconcileObjects - 资源协调

```go
// pkg/controller/trainjob_controller.go:147-158
func (r *TrainJobReconciler) reconcileObjects(ctx context.Context, runtime jobruntimes.Runtime, trainJob *trainer.TrainJob) error {
    // 调用 Runtime 生成资源配置
    objects, err := runtime.NewObjects(ctx, trainJob)
    if err != nil {
        return err
    }
    // 使用 Server-Side Apply 应用资源
    for _, object := range objects {
        if err := r.client.Apply(ctx, object, client.FieldOwner("trainer"), client.ForceOwnership); err != nil {
            return err
        }
    }
    return nil
}
```

**关键点**:
- 使用 **Server-Side Apply (SSA)** 而非传统的 Create/Update
- `client.FieldOwner("trainer")` 声明资源所有权
- `client.ForceOwnership` 强制覆盖冲突字段

### Controller 注册与启动

```go
// pkg/controller/trainjob_controller.go:237-255
func (r *TrainJobReconciler) SetupWithManager(mgr ctrl.Manager, options controller.Options) error {
    b := builder.TypedControllerManagedBy[reconcile.Request](mgr).
        Named("trainjob_controller").
        WithOptions(options).
        WatchesRawSource(source.TypedKind(
            mgr.GetCache(),
            &trainer.TrainJob{},
            &handler.TypedEnqueueRequestForObject[*trainer.TrainJob]{},
            r,  // TrainJobReconciler 也实现了 Predicate 接口
        ))

    // 注册各 Runtime 的事件处理器 (如 JobSet 变化时触发 TrainJob 协调)
    for _, runtime := range r.runtimes {
        for _, registrar := range runtime.EventHandlerRegistrars() {
            if registrar != nil {
                b = registrar(b, mgr.GetClient(), mgr.GetCache())
            }
        }
    }
    return b.Complete(r)
}
```

**关键设计**:
- `TrainJobReconciler` 同时实现了 `Reconciler` 和 `Predicate` 接口
- 各 Runtime 可以注册额外的 Watch (如 JobSet 变化触发 TrainJob 协调)

### TrainingRuntime Controller

```go
// pkg/controller/trainingruntime_controller.go
type TrainingRuntimeReconciler struct {
    client   client.Client
    recorder record.EventRecorder
}

func (r *TrainingRuntimeReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var runtime trainer.TrainingRuntime
    if err := r.client.Get(ctx, req.NamespacedName, &runtime); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // 检查 runtime 是否被废弃
    if _, deprecated := runtime.Labels[constants.LabelSupport]; deprecated {
        r.recorder.Event(&runtime, corev1.EventTypeWarning, "Deprecated", "This runtime is deprecated")
    }
    return ctrl.Result{}, nil
}
```

## 状态管理

### 条件类型

```go
// pkg/apis/trainer/v1alpha1/trainjob_types.go:57-66
const (
    TrainJobSuspended string = "Suspended"  // 暂停状态
    TrainJobComplete  string = "Complete"   // 完成状态
    TrainJobFailed    string = "Failed"     // 失败状态
)
```

### 状态同步逻辑

```go
// pkg/controller/trainjob_controller.go:226-235
func setTrainJobStatus(ctx context.Context, runtime jobruntimes.Runtime, trainJob *trainer.TrainJob) error {
    // 从 Runtime (JobSet) 获取状态
    status, err := runtime.TrainJobStatus(ctx, trainJob)
    if err != nil {
        return err
    }
    if status != nil {
        trainJob.Status = *status
    }
    return nil
}
```

## 并发与一致性

- 使用 controller-runtime 的 Work Queue 保证同一资源的协调串行化
- 使用 Server-Side Apply 处理并发更新冲突
- 通过 `metav1.Condition` 的 `ObservedGeneration` 跟踪资源版本
