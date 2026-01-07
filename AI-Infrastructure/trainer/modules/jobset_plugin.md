# JobSet Plugin

## 职责

JobSet 插件是 Trainer 的核心资源构建插件，负责：

1. 构建 JobSet 资源 (Kubernetes 原生工作负载)
2. 识别 Pod 网络端点
3. 同步 TrainJob 状态
4. 验证 TrainJob 配置

## 核心接口实现

```go
// pkg/runtime/framework/plugins/jobset/jobset.go
type JobSet struct {
    client     client.Client
    restMapper meta.RESTMapper
    scheme     *apiruntime.Scheme
    logger     logr.Logger
}

var _ framework.WatchExtensionPlugin = (*JobSet)(nil)
var _ framework.PodNetworkPlugin = (*JobSet)(nil)
var _ framework.ComponentBuilderPlugin = (*JobSet)(nil)
var _ framework.TrainJobStatusPlugin = (*JobSet)(nil)
var _ framework.CustomValidationPlugin = (*JobSet)(nil)

const Name = constants.JobSetKind  // "JobSet"
```

## 交互关系

**调用方**:
- Plugin Framework (PodNetwork, Build, Status 阶段)

**依赖方**:
- JobSet API (`sigs.k8s.io/jobset`)
- `runtime.Info` 对象
- `apply` 工具包

## 源码走读记录

### IdentifyPodNetwork - 识别 Pod 网络端点

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:208-235
func (j *JobSet) IdentifyPodNetwork(info *runtime.Info, trainJob *trainer.TrainJob) error {
    if info == nil || trainJob == nil {
        return nil
    }

    spec, ok := runtime.TemplateSpecApply[jobsetv1alpha2ac.JobSetSpecApplyConfiguration](info)
    if !ok {
        return nil
    }

    // 确定 subdomain (用于 DNS)
    subDomain := trainJob.Name
    if jobSetNet := spec.Network; jobSetNet != nil && jobSetNet.Subdomain != nil {
        subDomain = *jobSetNet.Subdomain
    }

    // 为每个 ReplicatedJob 生成端点
    for rJobIdx, rJob := range spec.ReplicatedJobs {
        podCount := info.TemplateSpec.PodSets[rJobIdx].Count
        rJobReplicas := constants.DefaultJobReplicas  // 1

        // 设置端点迭代器
        info.TemplateSpec.PodSets[rJobIdx].Endpoints = func(yield func(string) bool) {
            for podIdx := range ptr.Deref(podCount, 1) {
                // 生成 DNS 名称: <trainjob>-<replicatedjob>-<replica>-<podindex>.<subdomain>
                endpoint := fmt.Sprintf("%s-%s-%d-%d.%s",
                    trainJob.Name, *rJob.Name, rJobReplicas-1, podIdx, subDomain)
                if !yield(endpoint) {
                    return
                }
            }
        }
    }
    return nil
}
```

**端点命名规则**:
```
<trainjob-name>-<job-name>-<replica-index>-<pod-index>.<subdomain>

示例:
my-training-node-0-0.my-training
my-training-node-0-1.my-training
my-training-node-0-2.my-training
```

### Build - 构建 JobSet 资源

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:237-307
func (j *JobSet) Build(ctx context.Context, info *runtime.Info, trainJob *trainer.TrainJob) ([]apiruntime.ApplyConfiguration, error) {
    if info == nil || trainJob == nil {
        return nil, fmt.Errorf("runtime info or object is missing")
    }

    // 1. 检查是否需要更新 JobSet
    oldJobSet := &jobsetv1alpha2.JobSet{}
    if err := j.client.Get(ctx, client.ObjectKeyFromObject(trainJob), oldJobSet); err != nil {
        if !apierrors.IsNotFound(err) {
            return nil, err
        }
        oldJobSet = nil
    }

    // 如果 JobSet 已存在且未暂停，不进行更新
    if oldJobSet != nil &&
        !ptr.Deref(trainJob.Spec.Suspend, false) &&
        !ptr.Deref(oldJobSet.Spec.Suspend, false) {
        return nil, nil
    }

    // 2. 获取 JobSet 模板
    jobSetSpec, ok := runtime.TemplateSpecApply[jobsetv1alpha2ac.JobSetSpecApplyConfiguration](info)
    if !ok {
        return nil, nil
    }

    // 3. 应用来自 Info 的配置到 JobSet 模板
    for psIdx, ps := range info.TemplateSpec.PodSets {
        // 设置 parallelism 和 completions
        if ps.Count != nil {
            jobSetSpec.ReplicatedJobs[psIdx].Template.Spec.Parallelism = ps.Count
            jobSetSpec.ReplicatedJobs[psIdx].Template.Spec.Completions = ps.Count
        }

        // 合并 Volumes
        apply.UpsertVolumes(&jobSetSpec.ReplicatedJobs[psIdx].Template.Spec.Template.Spec.Volumes, ps.Volumes...)

        // 合并容器配置
        for containerIdx, container := range ps.Containers {
            apply.UpsertEnvVars(
                &jobSetSpec.ReplicatedJobs[psIdx].Template.Spec.Template.Spec.Containers[containerIdx].Env,
                container.Env...,
            )
            apply.UpsertPort(
                &jobSetSpec.ReplicatedJobs[psIdx].Template.Spec.Template.Spec.Containers[containerIdx].Ports,
                container.Ports...,
            )
            apply.UpsertVolumeMounts(
                &jobSetSpec.ReplicatedJobs[psIdx].Template.Spec.Template.Spec.Containers[containerIdx].VolumeMounts,
                container.VolumeMounts...,
            )
        }
    }

    // 4. 使用 Builder 构建最终的 JobSet
    jobSetBuilder := NewBuilder(jobsetv1alpha2ac.JobSet(trainJob.Name, trainJob.Namespace).
        WithLabels(maps.Clone(info.Labels)).
        WithAnnotations(maps.Clone(info.Annotations)).
        WithSpec(jobSetSpec))

    jobSet := jobSetBuilder.
        Initializer(trainJob).
        Trainer(info, trainJob).
        PodLabels(info.Scheduler.PodLabels).
        PodAnnotations(info.Scheduler.PodAnnotations).
        Suspend(trainJob.Spec.Suspend).
        Build().
        WithOwnerReferences(metav1ac.OwnerReference().
            WithAPIVersion(trainer.GroupVersion.String()).
            WithKind(trainer.TrainJobKind).
            WithName(trainJob.Name).
            WithUID(trainJob.UID).
            WithController(true).
            WithBlockOwnerDeletion(true))

    return []apiruntime.ApplyConfiguration{jobSet}, nil
}
```

### Status - 同步 TrainJob 状态

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:309-339
func (j *JobSet) Status(ctx context.Context, trainJob *trainer.TrainJob) (*trainer.TrainJobStatus, error) {
    // 获取 JobSet 状态
    jobSet := &jobsetv1alpha2.JobSet{}
    if err := j.client.Get(ctx, client.ObjectKeyFromObject(trainJob), jobSet); err != nil {
        return nil, err
    }

    status := trainJob.Status.DeepCopy()

    // 映射 JobSet 条件到 TrainJob 条件
    if completed := meta.FindStatusCondition(jobSet.Status.Conditions, string(jobsetv1alpha2.JobSetCompleted)); completed != nil && completed.Status == metav1.ConditionTrue {
        completed.Type = trainer.TrainJobComplete
        meta.SetStatusCondition(&status.Conditions, *completed)
    }
    if failed := meta.FindStatusCondition(jobSet.Status.Conditions, string(jobsetv1alpha2.JobSetFailed)); failed != nil && failed.Status == metav1.ConditionTrue {
        failed.Type = trainer.TrainJobFailed
        meta.SetStatusCondition(&status.Conditions, *failed)
    }

    // 同步 ReplicatedJob 状态
    var statuses []trainer.JobStatus
    for _, rjStatus := range jobSet.Status.ReplicatedJobsStatus {
        statuses = append(statuses, trainer.JobStatus{
            Name:      rjStatus.Name,
            Ready:     &rjStatus.Ready,
            Succeeded: &rjStatus.Succeeded,
            Failed:    &rjStatus.Failed,
            Active:    &rjStatus.Active,
            Suspended: &rjStatus.Suspended,
        })
    }
    status.JobsStatus = statuses

    return status, nil
}
```

### Validate - 验证配置

```go
// pkg/runtime/framework/plugins/jobset/jobset.go:86-152
func (j *JobSet) Validate(ctx context.Context, info *runtime.Info, oldObj, newObj *trainer.TrainJob) (admission.Warnings, field.ErrorList) {
    var allErrs field.ErrorList

    jobSetSpec, ok := runtime.TemplateSpecApply[jobsetv1alpha2ac.JobSetSpecApplyConfiguration](info)
    if !ok {
        return nil, nil
    }

    // 构建 ReplicatedJob -> 容器名称 映射
    rJobContainerNames := make(map[string]sets.Set[string])
    for _, rJob := range jobSetSpec.ReplicatedJobs {
        rJobContainerNames[*rJob.Name] = sets.New[string]()
        for _, c := range rJob.Template.Spec.Template.Spec.InitContainers {
            rJobContainerNames[*rJob.Name].Insert(*c.Name)
        }
        for _, c := range rJob.Template.Spec.Template.Spec.Containers {
            rJobContainerNames[*rJob.Name].Insert(*c.Name)
        }
    }

    // 验证 Initializer.Dataset
    if newObj.Spec.Initializer != nil && newObj.Spec.Initializer.Dataset != nil {
        if containers, ok := rJobContainerNames[constants.DatasetInitializer]; !ok {
            allErrs = append(allErrs, field.Invalid(runtimeRefPath, newObj.Spec.RuntimeRef,
                "must have dataset-initializer job when trainJob is configured with datasetConfig"))
        } else if !containers.Has(constants.DatasetInitializer) {
            allErrs = append(allErrs, field.Invalid(runtimeRefPath, newObj.Spec.RuntimeRef,
                "must have container with name dataset-initializer in the dataset-initializer job"))
        }
    }

    // 验证 PodTemplateOverrides
    for _, override := range newObj.Spec.PodTemplateOverrides {
        for _, targetJob := range override.TargetJobs {
            containers, ok := rJobContainerNames[targetJob.Name]
            if !ok {
                allErrs = append(allErrs, field.Invalid(podTemplateOverridePath, newObj.Spec.PodTemplateOverrides,
                    "must not have targetJob that doesn't exist in runtime"))
            }

            // 检查容器覆盖
            if override.Spec != nil {
                for _, overrideContainer := range override.Spec.Containers {
                    if !containers.Has(overrideContainer.Name) {
                        allErrs = append(allErrs, field.Invalid(podTemplateOverridePath, newObj.Spec.PodTemplateOverrides,
                            fmt.Sprintf("must not have container that doesn't exist in job %s", targetJob.Name)))
                    }
                    // 保留容器不允许通过 override 设置 env
                    if len(overrideContainer.Env) > 0 &&
                        (overrideContainer.Name == constants.DatasetInitializer ||
                         overrideContainer.Name == constants.ModelInitializer ||
                         overrideContainer.Name == constants.Node) {
                        allErrs = append(allErrs, field.Invalid(podTemplateOverridePath, newObj.Spec.PodTemplateOverrides,
                            "must not have envs for reserved containers"))
                    }
                }
            }
        }
    }

    // 验证 PodTemplateOverrides 不可变性
    allErrs = append(allErrs, j.checkPodTemplateOverridesImmutability(ctx, oldObj, newObj)...)

    return nil, allErrs
}
```

## JobSet Builder

```go
// pkg/runtime/framework/plugins/jobset/builder.go
type Builder struct {
    jobSet *jobsetv1alpha2ac.JobSetApplyConfiguration
}

func (b *Builder) Initializer(trainJob *trainer.TrainJob) *Builder {
    // 设置 dataset-initializer 和 model-initializer
}

func (b *Builder) Trainer(info *runtime.Info, trainJob *trainer.TrainJob) *Builder {
    // 设置 trainer 容器配置
    // 包括 image, command, args, env, resources
}

func (b *Builder) PodLabels(labels map[string]string) *Builder {
    // 设置 Pod 标签
}

func (b *Builder) Suspend(suspend *bool) *Builder {
    // 设置暂停状态
}

func (b *Builder) Build() *jobsetv1alpha2ac.JobSetApplyConfiguration {
    return b.jobSet
}
```

## 生成的 JobSet 结构

```yaml
apiVersion: jobset.x-k8s.io/v1alpha2
kind: JobSet
metadata:
  name: my-training
  namespace: default
  ownerReferences:
    - apiVersion: trainer.kubeflow.org/v1alpha1
      kind: TrainJob
      name: my-training
      controller: true
      blockOwnerDeletion: true
spec:
  suspend: false
  replicatedJobs:
    - name: node
      replicas: 1
      template:
        spec:
          parallelism: 4      # numNodes
          completions: 4      # numNodes
          template:
            metadata:
              labels:
                trainer.kubeflow.org/trainjob-ancestor-step: trainer
            spec:
              containers:
                - name: node
                  image: pytorch/pytorch:2.0.0
                  command: ["torchrun"]
                  args: ["--nnodes=4", "train.py"]
                  env:
                    - name: PET_NNODES
                      value: "4"
                    - name: PET_NPROC_PER_NODE
                      value: "8"
                  ports:
                    - containerPort: 29500
                  resources:
                    limits:
                      nvidia.com/gpu: 8
  network:
    subdomain: my-training
```

## 状态映射

| JobSet 条件 | TrainJob 条件 |
|:------------|:--------------|
| `JobSetCompleted` | `Complete` |
| `JobSetFailed` | `Failed` |

```go
// JobsStatus 映射
type JobStatus struct {
    Name      string  // ReplicatedJob 名称
    Ready     *int32  // 就绪 Pod 数
    Succeeded *int32  // 成功 Pod 数
    Failed    *int32  // 失败 Pod 数
    Active    *int32  // 活跃 Pod 数
    Suspended *int32  // 暂停 Pod 数
}
```

## Watch 扩展

```go
func (j *JobSet) ReconcilerBuilders() []runtime.ReconcilerBuilder {
    return []runtime.ReconcilerBuilder{
        func(b *builder.Builder, cl client.Client, cache cache.Cache) *builder.Builder {
            return b.Watches(
                &jobsetv1alpha2.JobSet{},
                handler.EnqueueRequestForOwner(
                    j.client.Scheme(), j.client.RESTMapper(),
                    &trainer.TrainJob{}, handler.OnlyControllerOwner(),
                ),
            )
        },
    }
}
```

当 JobSet 状态变化时，会触发其 Owner TrainJob 的协调，从而同步状态。
