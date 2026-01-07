# 闭环设计 (Reconcile Loops)

> 能演进的系统是控制回路：观察 → 比较 → 纠偏 → 再观察。

## 闭环 vs 流程

| 特性 | Trainer 的闭环 | 传统流程 |
|:-----|:---------------|:---------|
| 执行模式 | 持续 Watch + Reconcile | 一次性创建 |
| 容错 | 自动重试，最终一致 | 需要手动补偿 |
| 新增组件 | 自动适配（重新 Reconcile） | 需要修改流程 |
| 迁移 | 渐进式（逐个 TrainJob） | 通常全量 |
| 状态 | 期望状态 (TrainJob) + 当前状态 (JobSet) | 中间状态 |

---

## 核心闭环清单

| 闭环名称 | 触发方式 | 收敛周期 | 幂等性 | 源码位置 |
|:---------|:---------|:---------|:-------|:---------|
| TrainJob Reconcile | Watch + Resync | 秒级 | Yes (SSA) | `pkg/controller/trainjob_controller.go` |
| JobSet Reconcile | Watch | 秒级 | Yes | (外部: jobset-controller) |
| Status Sync | Watch JobSet | 秒级 | Yes | `pkg/controller/trainjob_controller.go` |

---

## Reconcile Loop 1: TrainJob Controller

### 闭环模型

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TrainJob Reconcile Loop                          │
│                                                                     │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐ │
│   │  Watch   │───▶│  Compare │───▶│  Build   │───▶│  Apply (SSA) │ │
│   │ TrainJob │    │ Runtime  │    │  Objects │    │  to Cluster  │ │
│   └──────────┘    └──────────┘    └──────────┘    └──────────────┘ │
│        ▲                                               │            │
│        │          ┌──────────────────────────────────┘            │
│        │          ▼                                                │
│        │    ┌──────────────┐                                       │
│        └────│  Sync Status │                                       │
│             │  (TrainJob)  │                                       │
│             └──────────────┘                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 触发条件

- [x] TrainJob 变更事件 (Watch)
- [x] 定时触发 (Resync，默认 10 小时)
- [x] JobSet 变更事件（通过 Watch 关联资源）
- [ ] 手动触发 (Force Reconcile) - 不支持

### 期望状态来源

**来源**: TrainJob CR + TrainingRuntime CR

```yaml
# TrainJob (用户定义的期望)
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: pytorch-job
spec:
  runtimeRef:
    name: torch-distributed  # 引用 Runtime
  trainer:
    image: pytorch/pytorch:2.0
    numNodes: 4
```

```yaml
# TrainingRuntime (平台提供的模板)
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainingRuntime
metadata:
  name: torch-distributed
spec:
  mlPolicy:
    torch:
      numProcPerNode: "8"
  template:
    spec:
      replicatedJobs:
        - name: worker
          template: ...
```

### 当前状态获取

**观察逻辑**:
```go
// pkg/controller/trainjob_controller.go:98-115
func (r *TrainJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取 TrainJob
    var trainJob trainer.TrainJob
    if err := r.client.Get(ctx, req.NamespacedName, &trainJob); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 获取 Runtime
    runtimeObj := r.runtimeRegistry.Get(...)
    runtimeSpec, err := runtimeObj.GetRuntimeSpec(ctx, &trainJob)

    // 3. 获取现有 JobSet（如果存在）
    var jobSet jobsetv1alpha2.JobSet
    r.client.Get(ctx, types.NamespacedName{...}, &jobSet)
}
```

### 比较逻辑

**核心**: 不做显式比较，依赖 Server-Side Apply 的 3-way merge

```go
// 不需要手动比较，SSA 会自动处理
// 只需要声明期望状态，Apply 时自动计算 diff
func (r *TrainJobReconciler) applyObjects(ctx context.Context, objs []client.Object) error {
    for _, obj := range objs {
        // SSA 自动比较并 patch
        if err := r.client.Patch(ctx, obj, client.Apply,
            client.FieldOwner(FieldManager),
            client.ForceOwnership,
        ); err != nil {
            return err
        }
    }
    return nil
}
```

### 纠偏动作

| 差异类型 | 纠偏动作 | 幂等保证 |
|:---------|:---------|:---------|
| JobSet 不存在 | 创建 JobSet | SSA Create |
| JobSet spec 不匹配 | 更新 JobSet | SSA Patch |
| Secret 不存在 (MPI) | 创建 SSH Secret | SSA + Immutable |
| ConfigMap 不存在 (MPI) | 创建 hostfile | SSA |
| 状态不同步 | 更新 TrainJob status | 覆盖写入 |

### 收敛策略

**超时处理**:
- 单次协调超时: 无硬性限制（依赖 K8s client 超时）
- 整体收敛超时: 无限（持续重试直到成功）

**退避策略**:
```go
// controller-runtime 默认退避
// 首次失败: 立即重试
// 后续失败: 指数退避 (1s, 2s, 4s, ..., 最大 16 分钟)
ctrl.NewControllerManagedBy(mgr).
    WithOptions(controller.Options{
        RateLimiter: workqueue.DefaultControllerRateLimiter(),
    })
```

**死信/隔离**:
- 无专门的死信队列
- 持续失败会被指数退避限制频率
- 依赖监控告警发现问题

*源码位置*: `pkg/controller/trainjob_controller.go:84-166`

---

## Reconcile Loop 2: Status Sync

### 闭环模型

```
┌─────────────────────────────────────────────────────────────────┐
│                    Status Sync Loop                              │
│                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │  Watch   │───▶│  Extract │───▶│  Map     │───▶│  Update  │ │
│   │  JobSet  │    │  Status  │    │  to      │    │ TrainJob │ │
│   │          │    │          │    │ TrainJob │    │  Status  │ │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 状态映射

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:309-339
func (j *JobSet) SyncTrainJobStatusWithJobSet(
    ctx context.Context, trainJob *trainer.TrainJob, objs []client.Object,
) (*trainer.TrainJobStatus, error) {
    // 从 objs 中找到 JobSet
    for _, obj := range objs {
        if jobSet, ok := obj.(*jobsetv1alpha2.JobSet); ok {
            // 映射状态条件
            if suspended := meta.FindStatusCondition(...); suspended != nil {
                meta.SetStatusCondition(&status.Conditions, *suspended)
            }
            if completed := meta.FindStatusCondition(...); completed != nil {
                completed.Type = trainer.TrainJobComplete  // 重命名
                meta.SetStatusCondition(&status.Conditions, *completed)
            }
            if failed := meta.FindStatusCondition(...); failed != nil {
                failed.Type = trainer.TrainJobFailed
                meta.SetStatusCondition(&status.Conditions, *failed)
            }
        }
    }
    return &status, nil
}
```

### 状态条件映射表

| JobSet Condition | TrainJob Condition | 含义 |
|:-----------------|:-------------------|:-----|
| `JobSetSuspended` | `TrainJobSuspended` | 任务暂停 |
| `JobSetCompleted` | `TrainJobComplete` | 训练完成 |
| `JobSetFailed` | `TrainJobFailed` | 训练失败 |

---

## 幂等性保证

### Checklist

- [x] 创建操作使用唯一标识符（JobSet 名称 = TrainJob 名称）
- [x] 更新操作使用 SSA（自动处理版本冲突）
- [x] 删除操作通过 OwnerReference（GC 自动清理）
- [x] 副作用操作有去重机制（SSH 密钥使用 Immutable Secret）

### 幂等性测试

| 场景 | 输入 | 首次执行 | 重复执行 | 预期结果 |
|:-----|:-----|:---------|:---------|:---------|
| 创建 TrainJob | 相同 spec | 创建 JobSet | 无操作 (SSA) | 一个 JobSet |
| 更新 TrainJob | 修改 numNodes | 更新 JobSet | 无操作 (SSA) | spec 已更新 |
| 删除 TrainJob | - | 级联删除 JobSet | 无操作 | 资源已清理 |

---

## 失败处理

### 失败分类

| 失败类型 | 可重试 | 处理策略 |
|:---------|:-------|:---------|
| 临时错误 (网络) | Yes | 指数退避重试 |
| 资源冲突 (SSA) | Yes | 自动重试 (SSA 处理) |
| Runtime 不存在 | No | 设置失败状态，停止重试 |
| 验证错误 | No | Webhook 拒绝，不进入 Reconcile |
| 配额不足 | Wait | 等待资源释放后自动恢复 |

### 错误处理代码

```go
// pkg/controller/trainjob_controller.go:118-130
runtimeSpec, err := runtimeObj.GetRuntimeSpec(ctx, &trainJob)
if err != nil {
    if apierrors.IsNotFound(err) {
        // Runtime 不存在，设置失败状态
        trainJob.Status.Conditions = append(trainJob.Status.Conditions,
            metav1.Condition{
                Type:    trainer.TrainJobFailed,
                Status:  metav1.ConditionTrue,
                Reason:  "RuntimeNotFound",
                Message: err.Error(),
            })
        return ctrl.Result{}, r.client.Status().Update(ctx, &trainJob)
    }
    // 其他错误，重试
    return ctrl.Result{}, err
}
```

### 人工介入路径

1. **查看状态**: `kubectl get trainjob <name> -o yaml`
2. **查看事件**: `kubectl describe trainjob <name>`
3. **查看日志**: `kubectl logs -l app=trainer-controller-manager`
4. **手动修复**: 修改 TrainJob spec 触发重新 Reconcile

---

## 可观测性

### 关键指标

| 指标 | 类型 | 含义 |
|:-----|:-----|:-----|
| `controller_runtime_reconcile_total` | Counter | 协调总次数 |
| `controller_runtime_reconcile_time_seconds` | Histogram | 协调耗时 |
| `controller_runtime_reconcile_errors_total` | Counter | 协调错误次数 |
| `workqueue_depth` | Gauge | 待协调队列深度 |
| `workqueue_adds_total` | Counter | 入队总数 |
| `workqueue_retries_total` | Counter | 重试总数 |

### 关键日志

```
level=info msg="Reconciling TrainJob" trainjob=default/pytorch-job
level=info msg="Building objects" runtime=torch-distributed
level=info msg="Applying JobSet" jobset=default/pytorch-job
level=info msg="Status synced" trainjob=default/pytorch-job complete=true
level=error msg="Reconcile failed" trainjob=default/pytorch-job error="runtime not found"
```

---

## 并发与冲突

### 并发控制

- [x] 单资源串行协调 (Work Queue)
  - 同一个 TrainJob 的多个事件被合并
  - controller-runtime 保证单 key 串行处理
- [x] 乐观并发 (Server-Side Apply)
  - 无需手动检查 ResourceVersion
  - 冲突时自动 merge
- [ ] 分布式锁 (Leader Election)
  - Controller 使用 Leader Election
  - 但单个资源不需要额外锁

### 冲突解决

**SSA 冲突解决**:
```go
// ForceOwnership=true 表示即使字段被其他 FieldManager 拥有也强制覆盖
client.Patch(ctx, obj, client.Apply,
    client.FieldOwner(FieldManager),
    client.ForceOwnership,  // 关键：强制获取字段所有权
)
```

**并发 Reconcile 场景**:
```
Time    Controller Instance 1       Controller Instance 2
─────────────────────────────────────────────────────────
t0      收到 TrainJob 事件           (Leader Election，只有一个活跃)
t1      Get TrainJob
t2      Build JobSet
t3      Apply JobSet (SSA)
t4      更新 Status
t5                                  (备用，等待 Leader 失败)
```

---

## 总结

Trainer 的 Reconcile Loop 设计体现了 Kubernetes Operator 的最佳实践：

1. **声明式**: 用户只描述期望状态，Controller 负责收敛
2. **幂等性**: SSA 保证重复执行安全
3. **最终一致**: 通过持续重试保证最终达到期望状态
4. **可观测**: 丰富的 metrics 和 events 支持监控告警
