# 不变量分析 (Invariants)

> 系统不是为了"实现功能"，而是为了在变化、故障、需求冲突中维持少数关键不变量。

## 核心不变量清单

| 编号 | 不变量 | 类型 | 破坏后果 | 降级策略 |
|:-----|:-------|:-----|:---------|:---------|
| INV-1 | TrainJob 与 JobSet 一对一映射 | 数据一致性 | 孤儿资源/重复创建 | OwnerReference 自动清理 |
| INV-2 | 保留环境变量不可被用户覆盖 | 抽象边界 | 训练进程通信失败 | 验证拒绝 |
| INV-3 | PodTemplateOverrides 仅在暂停时可修改 | 数据一致性 | Pod 配置不一致 | 验证拒绝 |
| INV-4 | MPI SSH 密钥不可变 | 安全边界 | 密钥篡改风险 | Secret Immutable |
| INV-5 | 单一 Runtime 解析结果 | 抽象边界 | 配置冲突 | 优先级规则 |

---

## Invariant 1: TrainJob 与 JobSet 一对一映射

### 描述

每个 TrainJob 有且只有一个对应的 JobSet，JobSet 的生命周期完全由 TrainJob 控制。

### 为什么必须成立？

- TrainJob 需要准确反映底层训练状态
- 避免孤儿 JobSet 占用集群资源
- 确保删除 TrainJob 时清理所有相关资源

### 如何保证？

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:Build（节选）
jobSet := jobsetv1alpha2ac.JobSet(trainJob.Name, trainJob.Namespace).
    WithLabels(maps.Clone(info.Labels)).
    WithAnnotations(maps.Clone(info.Annotations)).
    WithSpec(jobSetSpec).
    // OwnerReference 保证级联删除
    WithOwnerReferences(metav1ac.OwnerReference().
        WithAPIVersion(trainer.GroupVersion.String()).
        WithKind(trainer.TrainJobKind).
        WithName(trainJob.Name).
        WithUID(trainJob.UID).
        WithController(true).
        WithBlockOwnerDeletion(true))
```

**关键机制**:
- OwnerReference 设置 Controller=true
- JobSet 名称与 TrainJob 名称相同（命名空间内唯一）
- Server-Side Apply 确保幂等创建

### 破坏场景

- 手动创建同名 JobSet（被 OwnerReference 检查阻止）
- 控制器重启时部分创建（SSA 幂等性保证）

### 恢复策略

1. 孤儿 JobSet 会被 GC 清理（OwnerReference 机制）
2. 重复的 JobSet 创建会失败（名称冲突）

*关联代码位置*: `pkg/runtime/framework/plugins/jobset/jobset.go`

---

## Invariant 2: 保留环境变量不可被用户覆盖

### 描述

`PET_NNODES`、`PET_NPROC_PER_NODE`、`PET_MASTER_ADDR`、`PET_MASTER_PORT` 等环境变量由系统自动设置，用户不能在 TrainJob.Spec.Trainer.Env 中指定。

### 为什么必须成立？

这些环境变量控制 PyTorch 分布式训练的关键参数：
- `PET_NNODES`: 节点数量，必须与实际 Pod 数量一致
- `PET_MASTER_ADDR`: Master 地址，必须指向正确的 Headless Service
- 用户错误设置会导致训练进程无法互相发现

### 如何保证？

```go
// pkg/runtime/framework/plugins/torch/torch.go:Validate（节选）
func (t *Torch) Validate(_ context.Context, runtimeInfo *runtime.Info, _, newObj *trainer.TrainJob) (admission.Warnings, field.ErrorList) {
    torchEnvs := sets.New[string]()
    for _, env := range newObj.Spec.Trainer.Env {
        if constants.TorchRunReservedEnvNames.Has(env.Name) {
            torchEnvs.Insert(env.Name)
        }
    }
    if torchEnvs.Len() > 0 {
        allErrs = append(allErrs, field.Invalid(trainerEnvsPath, newObj.Spec.Trainer.Env,
            fmt.Sprintf("must not have reserved envs: %v", sets.List(torchEnvs))))
    }
}
```

### 破坏场景

- 绕过 Webhook 直接修改 etcd（需要 etcd 访问权限）
- Webhook 服务不可用时的创建请求

### 恢复策略

- 定期 Reconcile 时检测并修正（当前未实现）
- 依赖 Webhook 可用性保证

*关联代码位置*: `pkg/runtime/framework/plugins/torch/torch.go:55-94`

---

## Invariant 3: PodTemplateOverrides 仅在暂停时可修改

### 描述

TrainJob 的 `podTemplateOverrides` 只能在 `suspend=true` 且 JobSet 无活跃 Pod 时修改。

补充：`runtimeRef` 与 `managedBy` 在 CRD 层面标记为 **不可变**（XValidation），与 `suspend` 无关。

### 为什么必须成立？

- 运行中修改 Pod 模板会导致新旧 Pod 配置不一致
- 可能导致训练状态混乱（部分 Worker 配置不同）
- 违反 Kubernetes 不可变基础设施原则

### 如何保证？

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:154-186
if changed {
    if !suspended {
        allErrs = append(allErrs, field.Forbidden(podTemplateOverridePath,
            "PodTemplateOverrides can only be modified when the TrainJob is suspended"))
    } else {
        // 检查 JobSet 是否有活跃 Pod
        for _, replicatedJob := range jobSet.Status.ReplicatedJobsStatus {
            if replicatedJob.Active > 0 {
                allErrs = append(allErrs, field.Forbidden(podTemplateOverridePath,
                    fmt.Sprintf("PodTemplateOverrides cannot be modified when JobSet's ReplicatedJob %s is still active", replicatedJob.Name)))
            }
        }
    }
}
```

### 破坏场景

- Race condition: 检查通过后 Pod 被调度
- JobSet 状态同步延迟

### 恢复策略

- JobSet 控制器会拒绝非法的 Pod 模板变更
- 最终一致性由 Reconcile 保证

*关联代码位置*: `pkg/runtime/framework/plugins/jobset/jobset.go:154-186`

---

## Invariant 4: MPI SSH 密钥不可变

### 描述

为 MPI 训练生成的 SSH 密钥 Secret 设置为 `Immutable: true`，创建后不可修改。

### 为什么必须成立？

- 防止密钥被篡改，保证通信安全
- 所有 Worker Pod 使用相同密钥
- 简化密钥管理，避免轮换复杂性

### 如何保证？

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:buildSSHAuthSecret（节选）
return corev1ac.Secret(sshAuthSecretName(trainJob.Name), trainJob.Namespace).
    WithType(corev1.SecretTypeSSHAuth).
    WithData(map[string][]byte{
        corev1.SSHAuthPrivateKey:  privatePEM,
        constants.MPISSHPublicKey: ssh.MarshalAuthorizedKey(publicKey),
    }).
    WithImmutable(true). // 关键：不可变
    WithOwnerReferences(ownerRef), nil
```

### 破坏场景

- 删除并重建 Secret（会导致 Pod 通信失败）
- 直接修改 etcd

### 恢复策略

- 删除 TrainJob 并重建
- Secret 通过 OwnerReference 级联删除

*关联代码位置*: `pkg/runtime/framework/plugins/mpi/mpi.go:269-300`

---

## Invariant 5: 单一 Runtime 解析结果

### 描述

TrainJob 引用的 Runtime（TrainingRuntime 或 ClusterTrainingRuntime）必须存在且唯一，解析结果确定性。

### 为什么必须成立？

- Runtime 定义了 JobSet 模板、插件配置
- 不确定的解析会导致不可预测的训练配置

### 如何保证？

```go
// pkg/controller/trainjob_controller.go:Reconcile（节选）
runtimeRefGK := runtime.RuntimeRefToRuntimeRegistryKey(trainJob.Spec.RuntimeRef)
runtimeImpl := r.runtimes[runtimeRefGK]

// runtime.NewObjects() 内部会 client.Get 对应的 (Cluster)TrainingRuntime
objects, err := runtimeImpl.NewObjects(ctx, &trainJob)
```

**选择规则**：
- 由 `spec.runtimeRef.kind` 显式指定引用类型（默认 `ClusterTrainingRuntime`）
- `spec.runtimeRef` 字段本身不可变，因此同一个 TrainJob 的“runtime 身份”稳定

### 破坏场景

- 同名的 TrainingRuntime 和 ClusterTrainingRuntime
- Runtime 被删除但 TrainJob 仍在运行

### 恢复策略

- 创建时验证 Runtime 存在
- 运行时 Reconcile 检测 Runtime 缺失并报错

*关联代码位置*: `pkg/runtime/core/trainingruntime.go`, `pkg/runtime/core/clustertrainingruntime.go`

---

## 不变量验证

### 测试覆盖

| 不变量 | 单元测试 | 集成测试 | E2E 测试 |
|:-------|:---------|:---------|:---------|
| INV-1 | Yes | Yes | Yes |
| INV-2 | Yes | Yes | No |
| INV-3 | Yes | Yes | No |
| INV-4 | Yes | No | No |
| INV-5 | Yes | Yes | Yes |

### 运行时检查

- Webhook 验证器（INV-2, INV-3）
- Reconcile 状态检查（INV-1, INV-5）
- Kubernetes GC（INV-1, INV-4）

---

## 总结

| 不变量 | 重要程度 | 维护难度 | 当前状态 |
|:-------|:---------|:---------|:---------|
| INV-1 | 高 | 低 | 健康 |
| INV-2 | 高 | 低 | 健康 |
| INV-3 | 中 | 中 | 健康 |
| INV-4 | 高 | 低 | 健康 |
| INV-5 | 中 | 低 | 健康 |
