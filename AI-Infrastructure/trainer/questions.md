# 疑问与解答 (Q&A)

## 待解决

- [ ] （欢迎补充新问题）

## 已解决

### Q: 弹性训练 (Elastic Training) 如何实现？

**A: 当前代码仅提供了 API/Schema 字段，控制面尚未实现弹性伸缩。**

- `TorchElasticPolicy` 已在 CRD 类型中定义（包含 `minNodes/maxNodes/maxRestarts/metrics`）：`pkg/apis/trainer/v1alpha1/trainingruntime_types.go`
- Torch 插件里明确标注了待办：`pkg/runtime/framework/plugins/torch/torch.go` 的 TODO（需要 JobSet 支持 Elastic Jobs）
- 现状：Trainer 不会基于 `elasticPolicy.metrics` 创建 HPA，也不会把 `--nnodes=min:max` 之类的参数注入到 torchrun（仍以 `TrainJob.Spec.Trainer.NumNodes` 覆盖 Pod 数为主）

---

### Q: 多租户场景下，如何限制 TrainJob 只能引用特定的 ClusterTrainingRuntime？

**A: Trainer 目前没有“内建租户策略”字段/控制逻辑，通常需要借助 Kubernetes Admission/Policy。**

原因：TrainJob 的校验由 Trainer 的 webhook 执行（`pkg/webhooks/trainjob_webhook.go`），其检查 runtime 是否存在并执行插件校验，但不会按 namespace/tenant 做引用白名单控制。

可行做法（按常见落地路径排序）：
1. **集群侧策略**：用 `ValidatingAdmissionPolicy`（CEL）、Kyverno、OPA Gatekeeper 等在 admission 阶段拒绝不允许的 `.spec.runtimeRef.name/.spec.runtimeRef.kind`
2. **自定义 webhook**：实现一个额外的 validating webhook，根据 namespace/labels/annotations 维护“允许引用哪些 ClusterTrainingRuntime”的规则
3. **平台约定**：把 runtime 按租户拆为 namespaced `TrainingRuntime`（避免 cluster-scoped 共享），并通过 RBAC 控制谁能创建/修改这些 runtimes（但 RBAC 本身无法阻止“引用名字”，需要 admission 才能真正拒绝）

---

### Q: 如何处理训练过程中的 checkpoint 持久化？

**A: Trainer 不直接管理 checkpoint，靠 runtime 模板/PodTemplateOverrides 把持久化卷挂进训练容器。**

常见模式：
- 在 (Cluster)TrainingRuntime 的 JobSet 模板里声明 PVC/CSI 等 volume + mount（或用 `.spec.podTemplateOverrides` 给 TrainJob 动态追加）
- 训练脚本把输出目录指向该挂载点

TorchTune 场景的补充：Torch 插件会从 runtime 模板中提取并“固化”一部分关键 override（包含 checkpoint/output/tokenizer 等配置）并追加到命令行：
- 提取逻辑：`pkg/runtime/framework/plugins/torch/torchtune.go:extractOverridesFromRuntime`
- 固化的 key 集合：`pkg/constants/constants.go: TorchTuneImmutableConfigs`（含 `checkpointer.checkpoint_dir` 等）

---

### Q: 为什么 TrainJob 使用 JobSet 而不是直接创建 Job？

**A:**
JobSet 提供了以下 TrainJob 需要的能力：
1. **ReplicatedJobs**: 可以定义多组 Job，如 launcher + worker
2. **Startup Order**: 保证特定组 Job 先启动 (如 dataset-initializer)
3. **Failure Policy**: 统一的失败处理策略
4. **Headless Service**: 自动创建用于 Pod 间通信的 Service

*(关联代码位置: `pkg/runtime/framework/plugins/jobset/`)*

---

### Q: TrainJob 如何知道训练完成或失败？

**A:**
通过 JobSet 的状态同步：

1. JobSet 监控所有 ReplicatedJob 的状态
2. 当所有 Job 完成时，JobSet 设置 `JobSetCompleted` 条件
3. 当任何 Job 失败时，JobSet 设置 `JobSetFailed` 条件
4. TrainJob Controller 将这些条件映射到 TrainJob 状态

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:316-323
if completed := meta.FindStatusCondition(jobSet.Status.Conditions, string(jobsetv1alpha2.JobSetCompleted)); completed != nil && completed.Status == metav1.ConditionTrue {
    completed.Type = trainer.TrainJobComplete
    meta.SetStatusCondition(&status.Conditions, *completed)
}
```

*(关联代码位置: `pkg/runtime/framework/plugins/jobset/jobset.go:309-339`)*

---

### Q: MPI 训练如何保证 SSH 密钥安全？

**A:**
1. 使用 ECDSA P-521 椭圆曲线生成密钥对，安全性高
2. Secret 设置为 `Immutable: true`，防止被篡改
3. Secret 通过 OwnerReference 关联到 TrainJob，删除任务时自动清理
4. 只有同一 TrainJob 的 Pod 才能访问该 Secret

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:286-293
return corev1ac.Secret(sshAuthSecretName(trainJob.Name), trainJob.Namespace).
    WithType(corev1.SecretTypeSSHAuth).
    WithData(map[string][]byte{
        corev1.SSHAuthPrivateKey:  privatePEM,
        constants.MPISSHPublicKey: ssh.MarshalAuthorizedKey(publicKey),
    }).
    WithImmutable(true).
    WithOwnerReferences(ownerRef), nil
```

*(关联代码位置: `pkg/runtime/framework/plugins/mpi/mpi.go:269-300`)*

---

### Q: 为什么 Torch 插件有保留环境变量的检查？

**A:**
`PET_NNODES`、`PET_MASTER_ADDR` 等环境变量由 Trainer 根据 TrainJob 配置自动设置。如果用户手动设置这些变量，可能导致：
1. 与实际 Pod 数量不一致
2. Master 地址错误，导致进程无法通信

因此 Trainer 在验证阶段禁止用户设置这些保留环境变量。

```go
// pkg/runtime/framework/plugins/torch/torch.go:74-84
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
```

*(关联代码位置: `pkg/runtime/framework/plugins/torch/torch.go:55-94`)*

---

### Q: PodTemplateOverrides 为什么有不可变性检查？

**A:**
修改 PodTemplateOverrides 会导致 JobSet 的 Pod 模板变化。如果在训练运行中修改，可能导致：
1. 新旧 Pod 配置不一致
2. 训练状态混乱

因此只允许在 TrainJob 暂停 (suspend=true) 且 JobSet 无活跃 Pod 时修改。

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

*(关联代码位置: `pkg/runtime/framework/plugins/jobset/jobset.go:154-186`)*

---

### Q: ClusterTrainingRuntime 和 TrainingRuntime 有什么区别？

**A:**

| 特性 | ClusterTrainingRuntime | TrainingRuntime |
|:-----|:----------------------|:----------------|
| 作用域 | 集群级 | 命名空间级 |
| 可访问性 | 所有命名空间的 TrainJob | 仅同命名空间的 TrainJob |
| 适用场景 | 平台团队提供的标准模板 | 团队自定义模板 |

```go
// pkg/runtime/core/registry.go
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
```

*(关联代码位置: `pkg/runtime/core/registry.go`)*

---

### Q: 如何支持多种 Gang 调度器？

**A:**
通过 `PodGroupPolicy` 字段选择调度器：

```yaml
spec:
  podGroupPolicy:
    coscheduling:           # 使用 scheduler-plugins
      scheduleTimeoutSeconds: 300
    # 或者
    volcano:                # 使用 Volcano
      networkTopology: ...
```

对应的插件 (Coscheduling/Volcano) 会设置相应的 Pod 标签和调度器名称。

*(关联代码位置: `pkg/runtime/framework/plugins/coscheduling/`, `pkg/runtime/framework/plugins/volcano/`)*

---

### Q: numProcPerNode 设置为 "auto" 时如何工作？

**A:**
"auto" 是一个特殊值，会传递给 `torchrun` 命令：

```bash
torchrun --nproc-per-node=auto train.py
```

torchrun 会在运行时检测可用的 GPU 数量，并据此设置进程数。这在用户不确定 GPU 数量时非常有用。

如果配置了 CPU 训练 (无 GPU 资源)，Trainer 会将 "auto" 转换为 CPU 核数。

*(关联代码位置: `pkg/runtime/framework/plugins/torch/torch.go:120-122`)*
