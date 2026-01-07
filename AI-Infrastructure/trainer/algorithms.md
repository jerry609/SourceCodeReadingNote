# 关键算法 (Key Algorithms)

> 记录项目中使用的核心算法，包括其原理、复杂度、实现细节和应用场景。

## 算法分类

- **资源计算**: numProcPerNode 自动计算
- **拓扑生成**: Pod 端点生成、MPI hostfile 构建
- **状态同步**: Controller 协调循环
- **配置合并**: Apply Configuration 合并策略

---

## Algorithm 1: numProcPerNode 自动计算

### 分类
*资源计算 / 调度*

### 问题定义
根据用户配置和资源定义，自动确定每个训练节点应该运行的进程数 (通常等于 GPU 数)。

### 核心思想
优先使用用户显式配置，否则根据资源定义自动推断。支持 "auto"、"cpu"、"gpu" 等语义值。

### 算法描述

**输入**:
- `trainJob.Spec.Trainer.NumProcPerNode` (用户配置)
- `runtimePolicy.Torch.NumProcPerNode` (运行时默认值)
- `resourcesPerNode.Limits["nvidia.com/gpu"]` (GPU 资源)
- `resourcesPerNode.Requests["cpu"]` (CPU 资源)

**输出**:
- `numProcPerNode: intstr.IntOrString` (最终进程数)

**步骤**:
1. 如果 TrainJob 指定了 numProcPerNode，使用该值
2. 否则使用 RuntimePolicy 的默认值 (默认 "auto")
3. 如果值是 "cpu" 或 ("auto" 且无 GPU)：
   - 取 CPU 请求/限制的整数值
4. 如果值是 "gpu" 或 ("auto" 且有 GPU)：
   - 保持 "auto" 让 torchrun 自动检测

### 伪代码

```
function calculateNumProcPerNode(trainJob, runtimePolicy, resources):
    // 1. 确定初始值
    if trainJob.Trainer.NumProcPerNode != nil:
        numProc = trainJob.Trainer.NumProcPerNode
    else:
        numProc = runtimePolicy.Torch.NumProcPerNode ?? "auto"

    // 2. 获取资源信息
    gpuCount = resources.Limits["nvidia.com/gpu"]
    cpuCount = max(resources.Requests["cpu"], resources.Limits["cpu"])

    // 3. 解析语义值
    if numProc == "cpu" OR (numProc == "auto" AND gpuCount == 0):
        return max(1, cpuCount)

    return numProc  // "auto", "gpu", 或整数值
```

### 复杂度分析
- **时间复杂度**: O(1)
- **空间复杂度**: O(1)

### 实现代码

```go
// pkg/runtime/framework/plugins/torch/torch.go:97-123
func (t *Torch) EnforceMLPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error {
    numProcPerNode := ptr.Deref(info.RuntimePolicy.MLPolicySource.Torch.NumProcPerNode, intstr.FromString("auto"))
    if trainJob.Spec.Trainer != nil && trainJob.Spec.Trainer.NumProcPerNode != nil {
        numProcPerNode = ptr.Deref(trainJob.Spec.Trainer.NumProcPerNode, intstr.FromString("auto"))
    }

    resourcesPerNode := ptr.Deref(runtime.ExtractResourcePerNodeFromRuntime(info), corev1.ResourceRequirements{})
    if jobTrainer := trainJob.Spec.Trainer; jobTrainer != nil && jobTrainer.ResourcesPerNode != nil {
        resourcesPerNode = ptr.Deref(jobTrainer.ResourcesPerNode, corev1.ResourceRequirements{})
    }

    gpuQ := runtime.GetNumGPUPerNode(&resourcesPerNode)
    if numProcPerNode.String() == "cpu" || numProcPerNode.String() == "auto" && gpuQ == 0 {
        numProcPerNode = intstr.FromInt(max(1, getNumCPUPerNode(&resourcesPerNode)))
    }
    // ...
}
```

---

## Algorithm 2: MPI Hostfile 生成

### 分类
*拓扑生成*

### 问题定义
为 MPI 分布式训练生成 hostfile，包含所有训练节点的 FQDN 和 slots 数。

### 核心思想
遍历所有训练节点的端点，按 MPI 实现 (OpenMPI/Intel/MPICH) 格式化输出。

### 算法描述

**输入**:
- `info.TemplateSpec.PodSets` (Pod 集合信息)
- `mpiPolicy.NumProcPerNode` (每节点进程数)
- `mpiPolicy.RunLauncherAsNode` (launcher 是否参与训练)
- `mpiPolicy.MPIImplementation` (MPI 实现类型)

**输出**:
- `hostfile: string` (MPI hostfile 内容)

**步骤**:
1. 遍历所有 PodSet
2. 筛选出训练节点 (node 或 runLauncherAsNode 时的 launcher)
3. 对每个 PodSet，遍历其端点
4. 按 MPI 实现格式输出

### 伪代码

```
function buildHostfile(info, mpiPolicy):
    hostfile = ""
    slots = mpiPolicy.NumProcPerNode

    for ps in info.PodSets:
        if not isTrainingNode(ps, mpiPolicy.RunLauncherAsNode):
            continue

        switch mpiPolicy.MPIImplementation:
            case OpenMPI:
                for endpoint in ps.Endpoints:
                    hostfile += "${endpoint} slots=${slots}\n"
            case Intel:
                // Intel MPI 格式
            case MPICH:
                // MPICH 格式

    return hostfile

function isTrainingNode(ps, runLauncherAsNode):
    return (runLauncherAsNode AND ps.Name == "launcher") OR ps.Name == "node"
```

### 复杂度分析
- **时间复杂度**: O(n)，n 为总 Pod 数
- **空间复杂度**: O(n)，存储 hostfile 内容

### 实现代码

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:306-332
func (m *MPI) buildHostFileConfigMap(info *runtime.Info, trainJob *trainer.TrainJob) *corev1ac.ConfigMapApplyConfiguration {
    var hostFile bytes.Buffer
    runLauncherAsNode := ptr.Deref(info.RuntimePolicy.MLPolicySource.MPI.RunLauncherAsNode, false)
    slots := ptr.Deref(info.RuntimePolicy.MLPolicySource.MPI.NumProcPerNode, 1)

    for _, ps := range info.TemplateSpec.PodSets {
        if !isNode(runLauncherAsNode, ps) {
            continue
        }
        switch *info.RuntimePolicy.MLPolicySource.MPI.MPIImplementation {
        case trainer.MPIImplementationOpenMPI:
            for e := range ps.Endpoints {
                hostFile.WriteString(fmt.Sprintf("%s slots=%d\n", e, slots))
            }
        }
    }

    return corev1ac.ConfigMap(/*...*/).WithData(map[string]string{
        constants.MPIHostfileName: hostFile.String(),
    })
}
```

### 输出示例

```
# OpenMPI hostfile
my-job-node-0-0.my-job slots=8
my-job-node-0-1.my-job slots=8
my-job-node-0-2.my-job slots=8
my-job-node-0-3.my-job slots=8
```

---

## Algorithm 3: Apply Configuration 合并

### 分类
*配置合并*

### 问题定义
将多个来源的配置 (Runtime 模板、TrainJob 覆盖、插件注入) 合并成最终的 Kubernetes 资源配置。

### 核心思想
使用 "upsert" 模式：如果存在则更新，不存在则添加。对于 map 类型使用 key 匹配，对于 slice 类型使用 name 字段匹配。

### 算法描述

**输入**:
- `target: *[]T` (目标配置切片)
- `sources: ...T` (要合并的配置项)

**输出**:
- 修改后的 target

**步骤** (以 EnvVar 为例):
1. 对于每个 source env:
   - 在 target 中查找同名的 env
   - 如果找到，替换值
   - 如果未找到，追加到末尾

### 实现代码

```go
// pkg/apply/apply.go
func UpsertEnvVars(envs *[]corev1ac.EnvVarApplyConfiguration, newEnvs ...corev1ac.EnvVarApplyConfiguration) {
    for _, newEnv := range newEnvs {
        found := false
        for i, env := range *envs {
            if *env.Name == *newEnv.Name {
                (*envs)[i] = newEnv  // 替换
                found = true
                break
            }
        }
        if !found {
            *envs = append(*envs, newEnv)  // 追加
        }
    }
}

func UpsertVolumes(volumes *[]corev1ac.VolumeApplyConfiguration, newVolumes ...corev1ac.VolumeApplyConfiguration) {
    // 类似逻辑，以 Name 为 key
}

func UpsertVolumeMounts(mounts *[]corev1ac.VolumeMountApplyConfiguration, newMounts ...corev1ac.VolumeMountApplyConfiguration) {
    // 类似逻辑，以 Name 为 key
}
```

### 复杂度分析
- **时间复杂度**: O(n*m)，n 为目标长度，m 为源长度
- **空间复杂度**: O(1)，原地修改

### 优化空间
可以使用 map 建立索引，将时间复杂度降至 O(n+m)，但对于通常较小的配置列表，线性查找足够高效。

---

## Algorithm 4: Controller 协调循环

### 分类
*状态同步*

### 问题定义
确保 TrainJob 的期望状态与实际状态保持一致。

### 核心思想
Level-triggered (水平触发) 协调：每次协调时计算完整的期望状态，并应用差异。

### 算法描述

```
function Reconcile(trainJob):
    // 1. 获取当前状态
    currentJob = client.Get(trainJob)
    if currentJob not found:
        return

    // 2. 查找对应的 Runtime
    runtime = runtimes[currentJob.RuntimeRef]
    if runtime not found:
        setFailedCondition(currentJob, "runtime not supported")
        return

    // 3. 生成期望资源
    objects = runtime.NewObjects(currentJob)

    // 4. 应用资源 (SSA 自动处理差异)
    for object in objects:
        client.Apply(object)

    // 5. 同步状态
    status = runtime.TrainJobStatus(currentJob)
    if status changed:
        client.Status().Patch(currentJob)
```

### 关键特性

1. **幂等性**: 多次协调产生相同结果
2. **最终一致性**: 系统最终会收敛到期望状态
3. **事件驱动**: 由 Watch 事件触发，避免轮询
4. **乐观并发**: 使用 resourceVersion 避免冲突

### 复杂度分析
- **时间复杂度**: O(n)，n 为生成的资源数
- **空间复杂度**: O(n)，临时存储资源配置

---

## 算法总结表

| 算法 | 分类 | 核心场景 | 复杂度 | 备注 |
|:-----|:-----|:---------|:-------|:-----|
| numProcPerNode 计算 | 资源计算 | Torch 分布式训练 | O(1) | 支持 auto/cpu/gpu |
| MPI Hostfile 生成 | 拓扑生成 | MPI 分布式训练 | O(n) | 支持 OpenMPI |
| Apply Configuration 合并 | 配置合并 | 资源构建 | O(n*m) | Upsert 模式 |
| Controller 协调循环 | 状态同步 | 控制器核心 | O(n) | Level-triggered |
