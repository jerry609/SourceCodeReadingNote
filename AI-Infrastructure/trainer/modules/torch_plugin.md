# Torch Plugin

## 职责

Torch 插件负责配置 PyTorch 分布式训练环境，包括：

1. 设置 `torchrun` 所需的环境变量
2. 配置 TorchTune 微调参数
3. 验证训练配置的合法性

## 核心接口实现

```go
// pkg/runtime/framework/plugins/torch/torch.go
type Torch struct{}

var _ framework.EnforceMLPolicyPlugin = (*Torch)(nil)
var _ framework.CustomValidationPlugin = (*Torch)(nil)

const Name = "Torch"
```

## 交互关系

**调用方**:
- Plugin Framework (EnforceMLPolicy 阶段)

**依赖方**:
- `runtime.Info` 对象
- `apply` 工具包

## 源码走读记录

### EnforceMLPolicy - 设置 PyTorch 分布式环境

```go
// pkg/runtime/framework/plugins/torch/torch.go:97-189
func (t *Torch) EnforceMLPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error {
    // 检查是否是 Torch 运行时
    if info == nil || info.RuntimePolicy.MLPolicySource == nil || info.RuntimePolicy.MLPolicySource.Torch == nil {
        return nil
    }

    // 1. 获取 Trainer PodSet，设置节点数
    trainerPS := info.FindPodSetByAncestor(constants.AncestorTrainer)
    if trainerPS != nil && trainerPS.Count != nil && trainJob.Spec.Trainer != nil && trainJob.Spec.Trainer.NumNodes != nil {
        *trainerPS.Count = *trainJob.Spec.Trainer.NumNodes
    }

    // 2. 确定 numProcPerNode
    numProcPerNode := ptr.Deref(info.RuntimePolicy.MLPolicySource.Torch.NumProcPerNode, intstr.FromString("auto"))
    if trainJob.Spec.Trainer != nil && trainJob.Spec.Trainer.NumProcPerNode != nil {
        numProcPerNode = ptr.Deref(trainJob.Spec.Trainer.NumProcPerNode, intstr.FromString("auto"))
    }

    // 3. 根据资源配置自动计算 numProcPerNode
    resourcesPerNode := ptr.Deref(runtime.ExtractResourcePerNodeFromRuntime(info), corev1.ResourceRequirements{})
    gpuQ := runtime.GetNumGPUPerNode(&resourcesPerNode)

    // 如果是 "cpu" 或 "auto" 且无 GPU，则按 CPU 数量计算
    if numProcPerNode.String() == "cpu" || numProcPerNode.String() == "auto" && gpuQ == 0 {
        numProcPerNode = intstr.FromInt(max(1, getNumCPUPerNode(&resourcesPerNode)))
    }

    // 4. 找到 trainer 容器，设置环境变量
    trainerContainer := info.FindContainerByPodSetAncestorContainerName(constants.AncestorTrainer, constants.Node)
    if trainerContainer != nil {
        // 合并用户自定义环境变量
        apply.UpsertEnvVars(&trainerContainer.Env, apply.EnvVars(trainJob.Spec.Trainer.Env...)...)

        // 添加 PyTorch 分布式 "PET_" 环境变量
        apply.UpsertEnvVars(&trainerContainer.Env,
            *corev1ac.EnvVar().
                WithName(constants.TorchEnvNumNodes).           // PET_NNODES
                WithValue(fmt.Sprintf("%d", numNodes)),
            *corev1ac.EnvVar().
                WithName(constants.TorchEnvNumProcPerNode).     // PET_NPROC_PER_NODE
                WithValue(numProcPerNode.String()),
            *corev1ac.EnvVar().
                WithName(constants.TorchEnvNodeRank).           // PET_NODE_RANK
                WithValueFrom(corev1ac.EnvVarSource().
                    WithFieldRef(corev1ac.ObjectFieldSelector().
                        WithFieldPath(constants.JobCompletionIndexFieldPath))),
        )

        // 添加 Master 地址和端口
        apply.UpsertEnvVars(&trainerContainer.Env,
            *corev1ac.EnvVar().
                WithName(constants.TorchEnvMasterAddr).         // PET_MASTER_ADDR
                WithValue(fmt.Sprintf("%s-%s-0-0.%s", trainJob.Name, constants.Node, trainJob.Name)),
            *corev1ac.EnvVar().
                WithName(constants.TorchEnvMasterPort).         // PET_MASTER_PORT
                WithValue(fmt.Sprintf("%d", constants.ContainerTrainerPort)),
        )

        // 添加容器端口
        apply.UpsertPort(&trainerContainer.Ports,
            *corev1ac.ContainerPort().WithContainerPort(constants.ContainerTrainerPort))
    }

    return nil
}
```

### PyTorch 分布式训练环境变量

| 环境变量 | 说明 | 示例值 |
|:---------|:-----|:-------|
| `PET_NNODES` | 训练节点总数 | `4` |
| `PET_NPROC_PER_NODE` | 每节点进程数 (GPU 数) | `8` |
| `PET_NODE_RANK` | 当前节点排名 | `0`, `1`, `2`, `3` |
| `PET_MASTER_ADDR` | Master 节点地址 | `my-job-node-0-0.my-job` |
| `PET_MASTER_PORT` | Master 节点端口 | `29500` |

这些环境变量会被 `torchrun` 自动读取：

```bash
# torchrun 会使用这些环境变量
torchrun --nnodes=$PET_NNODES \
         --nproc-per-node=$PET_NPROC_PER_NODE \
         --node-rank=$PET_NODE_RANK \
         --master-addr=$PET_MASTER_ADDR \
         --master-port=$PET_MASTER_PORT \
         train.py
```

### TorchTune 支持

```go
// pkg/runtime/framework/plugins/torch/torch.go
// 当命令是 "tune run" 时，处理 TorchTune 特殊逻辑
if slices.Equal(trainJob.Spec.Trainer.Command, constants.TorchTuneEntrypoint) {
    // 多节点或多设备训练需要配置 rendezvous 端点
    if numNodes > 1 || numProcPerNode.IntVal > 1 || gpuQ > 1 {
        newCommand = append(newCommand,
            fmt.Sprintf("%s=%s-%s-0-0.%s:%d",
                constants.TorchTuneArgRdzvEndpoint,
                trainJob.Name, constants.Node, trainJob.Name, constants.ContainerTrainerPort,
            ),
        )
    }

    // 自动选择 recipe 和 config
    recipe, config := getRecipeAndConfig(numNodes, numProcPerNode, gpuQ, trainJob)
    newCommand = append(newCommand, recipe, constants.TorchTuneArgConfig, config)

    // 提取运行时配置的路径
    newCommand = append(newCommand, extractOverridesFromRuntime(info)...)
    trainJob.Spec.Trainer.Command = append(trainJob.Spec.Trainer.Command, newCommand...)
}
```

### TorchTune Recipe 选择逻辑

```go
// pkg/runtime/framework/plugins/torch/torchtune.go
func getRecipeAndConfig(numNodes int32, numProcPerNode intstr.IntOrString, gpuQ int, trainJob *trainer.TrainJob) (string, string) {
    // 单节点单设备
    if numNodes == 1 && (numProcPerNode.IntVal == 1 || gpuQ <= 1) {
        return constants.TorchTuneFullFinetuneSingleDevice,
               modelName + constants.TorchTuneFullFinetuneSingleDeviceConfigSuffix
    }

    // 单节点多设备
    if numNodes == 1 {
        return constants.TorchTuneFullFinetuneDistributed,
               modelName + constants.TorchTuneFullFinetuneMultiDevicesConfigSuffix
    }

    // 多节点分布式
    return constants.TorchTuneFullFinetuneDistributed,
           modelName + constants.TorchTuneFullFinetuneMultiNodesConfigSuffix
}
```

| 场景 | Recipe | Config 后缀 |
|:-----|:-------|:------------|
| 单节点单 GPU | `full_finetune_single_device` | `_full_single_device` |
| 单节点多 GPU | `full_finetune_distributed` | `_full` |
| 多节点 | `full_finetune_distributed` | `_full_multinode` |

### 验证逻辑

```go
// pkg/runtime/framework/plugins/torch/torch.go:55-94
func (t *Torch) Validate(_ context.Context, runtimeInfo *runtime.Info, _, newObj *trainer.TrainJob) (admission.Warnings, field.ErrorList) {
    var allErrs field.ErrorList

    // 1. 验证 numProcPerNode 的值
    if numProcPerNode.Type == intstr.String {
        allowed := sets.New("auto", "cpu", "gpu")
        if !allowed.Has(numProcPerNode.StrVal) {
            allErrs = append(allErrs, field.Invalid(numProcPerNodePath, numProcPerNode,
                fmt.Sprintf("must have an int value or %v", sets.List(allowed))))
        }
    }

    // 2. 检查保留环境变量 (用户不能手动设置)
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

    // 3. TorchTune 特定验证
    if slices.Equal(newObj.Spec.Trainer.Command, constants.TorchTuneEntrypoint) {
        _, torchTuneErrs := validateTorchTune(runtimeInfo, newObj)
        allErrs = append(allErrs, torchTuneErrs...)
    }

    return nil, allErrs
}
```

**保留环境变量** (不允许用户设置):
- `PET_NNODES`
- `PET_NPROC_PER_NODE`
- `PET_NODE_RANK`
- `PET_MASTER_ADDR`
- `PET_MASTER_PORT`

## numProcPerNode 计算逻辑

```
┌────────────────────────────────────────────────────────────────────────┐
│                    numProcPerNode 决策流程                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────┐     TrainJob 指定?      ┌─────────────────────────┐  │
│  │  开始       │─────────Yes────────────▶│ 使用 TrainJob 配置       │  │
│  └─────────────┘                         └─────────────────────────┘  │
│         │                                                              │
│         No                                                             │
│         ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ 使用 RuntimePolicy.Torch.NumProcPerNode (默认 "auto")            │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│         │                                                              │
│         ▼                                                              │
│  ┌──────────────────┐    值 = "cpu"?      ┌───────────────────────┐  │
│  │ 检查值类型        │────────Yes────────▶│ 按 CPU 核数计算         │  │
│  └──────────────────┘                     └───────────────────────┘  │
│         │                                                              │
│         │ 值 = "auto"?                                                │
│         ▼                                                              │
│  ┌──────────────────┐    GPU = 0?         ┌───────────────────────┐  │
│  │ 检查 GPU 资源     │────────Yes────────▶│ 按 CPU 核数计算         │  │
│  └──────────────────┘                     └───────────────────────┘  │
│         │                                                              │
│         No                                                             │
│         ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ 使用原始值 (GPU 数量或指定的整数值)                               │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## 与 torchrun 的集成

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TrainJob -> torchrun 映射                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  TrainJob:                       torchrun:                             │
│  ┌───────────────────────┐      ┌───────────────────────────────────┐ │
│  │ trainer:              │      │ torchrun                          │ │
│  │   numNodes: 4         │ ──▶  │   --nnodes=4                      │ │
│  │   numProcPerNode: 8   │ ──▶  │   --nproc-per-node=8              │ │
│  │                       │      │   --node-rank=${JOB_COMPLETION_   │ │
│  │                       │      │               INDEX}               │ │
│  │                       │      │   --master-addr=job-node-0-0.job  │ │
│  │                       │      │   --master-port=29500             │ │
│  │   command: [train.py] │ ──▶  │   train.py                        │ │
│  └───────────────────────┘      └───────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```
