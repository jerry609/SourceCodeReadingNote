# 训练执行流程 (Training Execution)

本文关注“控制面把对象下发之后，数据面如何真正跑起来”，并补齐 Trainer 与 JobSet/调度器/训练进程的分工边界。

## 参与者

- Trainer 控制面：TrainJob Controller + Runtime/Plugins（本仓库 Go 代码）
- 外部控制器：
  - JobSet Controller（`sigs.k8s.io/jobset`）
  - Gang 调度（可选）：scheduler-plugins PodGroup / Volcano
- 数据面：Job/Pod 中的训练进程（torchrun/torchtune/mpirun…）

## 从对象到 Pod：谁负责什么

1) Trainer 生成并 SSA Apply（见 `flows/trainjob_reconcile.md`）
- JobSet（核心 workload）
- PodGroup（可选，gang 调度）
- Secret/ConfigMap（可选，MPI）

2) JobSet Controller 根据 JobSet spec 创建/更新：
- Jobs（每个 replicatedJob 一个 Job 模板）
- Pods（由 Job controller 创建）
- 网络相关资源（取决于 JobSet spec 的 `.spec.network` 配置）

Trainer 不直接创建 Pod；它只负责把“期望的 JobSet spec”声明出来。

## 关键机制 1：端点与 DNS（PodNetwork）

JobSet 插件会为每个 PodSet 写入 `Endpoints` 迭代器（`runtime.Info.TemplateSpec.PodSets[*].Endpoints`）：
- 默认 subdomain：`trainJob.Name`
- 若 runtime 模板指定 `.spec.network.subdomain` 则使用该值

端点格式（当前 JobSet replicas 固定为 1）：
```
<trainjob>-<replicatedjob>-0-<podIndex>.<subdomain>
```

这些端点被后续插件消费：
- MPI：生成 hostfile（每行一个 endpoint + slots）
- Torch：拼接 master address（默认指向 `...-node-0-0.<subdomain>`）

## 关键机制 2：Torch / TorchTune 数据面契约

Torch 插件在 `EnforceMLPolicy` 阶段为 trainer 容器注入：
- `PET_NNODES` / `PET_NPROC_PER_NODE`
- `PET_NODE_RANK`：来自 `metadata.annotations['batch.kubernetes.io/job-completion-index']`
- `PET_MASTER_ADDR` / `PET_MASTER_PORT`（非 torchtune 场景）
- 额外开放端口：`29500`（用于 headless service 通信）

torchrun/torchtune 在容器内读取这些 env/args 完成 rendezvous。

TorchTune（`command == ["tune","run"]`）场景的额外行为：
- Trainer 会在 admission 阶段做模型/节点数等约束校验（`torch/torchtune.go:validateTorchTune`）
- Torch 插件会“改写 TrainJob.Spec.Trainer.Command”，补齐：
  - rendezvous endpoint
  - 自动选择的 recipe/config
  - 从 runtime 模板提取的“不可变 overrides”（如 `output_dir`、`checkpointer.checkpoint_dir` 等）

## 关键机制 3：MPI 数据面契约

MPI 插件的典型输出包含：
- SSH Secret（不可变）：`<trainjob>-mpi-ssh-auth`
- hostfile ConfigMap：`<trainjob>-mpi-hostfile`

并在 `EnforceMLPolicy` 阶段为相关 PodSet 注入：
- Secret/ConfigMap volume + volumeMount（默认 `/root/.ssh` 与 `/etc/mpi`）
- OpenMPI 环境变量（hostfile 位置、slots、rsh 参数等）

hostfile 的内容来自 PodNetwork 阶段生成的 endpoints：
```
<endpoint> slots=<numProcPerNode>
```

## 关键机制 4：Gang 调度（PodGroupPolicy）

当 (Cluster)TrainingRuntime 配置了 `.spec.podGroupPolicy`：
- `EnforcePodGroupPolicy`：向 Pod template 注入 gang label（如 `pod-group.scheduling.x-k8s.io/name=<trainjob>`）
- `ComponentBuilder`：构建 PodGroup 资源
  - `minMember`：按 `PodSet.Count` 汇总
  - `minResources`：按 `PodSet.SinglePodRequests * Count` 汇总

是否实际“成团调度”由集群侧调度器插件决定；Trainer 只负责声明 PodGroup 及相关标签。

## 状态如何回传到 TrainJob

- JobSet controller 持续更新 `JobSet.Status`
- Trainer 在每次 Reconcile 里调用 `runtime.TrainJobStatus(...)`
  - 当前主要由 JobSet 插件实现：读取 JobSet 条件并映射到 TrainJob 条件（Complete/Failed 等）

## 与 suspend 的关系（为何很多变更需要先暂停）

JobSet 插件有一个保护性策略：
- 如果 JobSet 已存在且 TrainJob/JobSet 都未 suspend，则 `Build()` 直接返回 `nil`（不更新 JobSet）

因此：
- 运行中修改 TrainJob spec，很多情况下不会触发底层 JobSet spec 被更新
- 推荐的变更路径：`suspend=true` → 修改 spec/overrides → reconcile 更新 JobSet → `suspend=false`

