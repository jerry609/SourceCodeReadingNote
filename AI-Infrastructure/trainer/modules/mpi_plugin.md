# MPI Plugin

## 职责

MPI 插件负责配置 MPI (Message Passing Interface) 分布式训练环境，包括：

1. 生成 SSH 密钥对 (Secret)
2. 生成 MPI hostfile (ConfigMap)
3. 挂载 SSH 密钥和 hostfile 到 Pod
4. 设置 OpenMPI 相关环境变量

## 核心接口实现

```go
// pkg/runtime/framework/plugins/mpi/mpi.go
type MPI struct {
    client client.Client
    scheme *apiruntime.Scheme
}

var _ framework.CustomValidationPlugin = (*MPI)(nil)
var _ framework.EnforceMLPolicyPlugin = (*MPI)(nil)
var _ framework.WatchExtensionPlugin = (*MPI)(nil)
var _ framework.ComponentBuilderPlugin = (*MPI)(nil)

const Name = "MPI"
```

## 交互关系

**调用方**:
- Plugin Framework (EnforceMLPolicy, Build 阶段)

**依赖方**:
- Kubernetes API (创建 Secret, ConfigMap)
- `runtime.Info` 对象

## 源码走读记录

### EnforceMLPolicy - 配置 MPI 环境

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:107-225
func (m *MPI) EnforceMLPolicy(info *runtime.Info, trainJob *trainer.TrainJob) error {
    if info == nil || info.RuntimePolicy.MLPolicySource == nil || info.RuntimePolicy.MLPolicySource.MPI == nil {
        return nil
    }

    // 1. 设置节点数
    if trainJob.Spec.Trainer != nil && trainJob.Spec.Trainer.NumNodes != nil {
        if node := info.FindPodSetByName(constants.Node); node != nil && node.Count != nil {
            if ptr.Deref(info.RuntimePolicy.MLPolicySource.MPI.RunLauncherAsNode, false) {
                // 当 launcher 也作为训练节点时，需要减 1
                *node.Count = max(*trainJob.Spec.Trainer.NumNodes-1, 1)
            } else {
                *node.Count = *trainJob.Spec.Trainer.NumNodes
            }
        }
    }

    // 2. 设置 numProcPerNode (slots)
    if trainJob.Spec.Trainer != nil && trainJob.Spec.Trainer.NumProcPerNode != nil {
        info.RuntimePolicy.MLPolicySource.MPI.NumProcPerNode = ptr.To(int32(trainJob.Spec.Trainer.NumProcPerNode.IntValue()))
    } else if *info.RuntimePolicy.MLPolicySource.MPI.NumProcPerNode == 1 {
        // 如果默认是 1，检查是否有多 GPU
        resourcesPerNode := ptr.Deref(runtime.ExtractResourcePerNodeFromRuntime(info), corev1.ResourceRequirements{})
        if gpuQ := runtime.GetNumGPUPerNode(&resourcesPerNode); gpuQ > 1 {
            info.RuntimePolicy.MLPolicySource.MPI.NumProcPerNode = ptr.To(int32(gpuQ))
        }
    }

    // 3. 为 node 和 launcher PodSet 添加 Volume 和 VolumeMount
    for psIdx, ps := range info.TemplateSpec.PodSets {
        if ps.Name != constants.Node && ps.Name != constants.Launcher {
            continue
        }

        // 添加 SSH 密钥 Volume
        apply.UpsertVolumes(&info.TemplateSpec.PodSets[psIdx].Volumes,
            *corev1ac.Volume().
                WithName(constants.MPISSHAuthVolumeName).
                WithSecret(corev1ac.SecretVolumeSource().
                    WithSecretName(fmt.Sprintf("%s%s", trainJob.Name, constants.MPISSHAuthSecretSuffix)).
                    WithItems(
                        corev1ac.KeyToPath().WithKey(corev1.SSHAuthPrivateKey).WithPath("id_rsa"),
                        corev1ac.KeyToPath().WithKey(constants.MPISSHPublicKey).WithPath("id_rsa.pub"),
                        corev1ac.KeyToPath().WithKey(constants.MPISSHPublicKey).WithPath("authorized_keys"),
                    ),
                ),
        )

        // Launcher 还需要 hostfile Volume
        if ps.Name == constants.Launcher {
            apply.UpsertVolumes(&info.TemplateSpec.PodSets[psIdx].Volumes,
                *corev1ac.Volume().
                    WithName(constants.MPIHostfileVolumeName).
                    WithConfigMap(corev1ac.ConfigMapVolumeSource().
                        WithName(fmt.Sprintf("%s%s", trainJob.Name, constants.MPIHostfileConfigMapSuffix)).
                        WithItems(corev1ac.KeyToPath().WithKey("hostfile").WithPath("hostfile").WithMode(0444)),
                    ),
            )
        }

        // 为容器添加 VolumeMount 和环境变量
        for cIdx, container := range ps.Containers {
            if container.Name != constants.Node {
                continue
            }

            // 挂载 SSH 密钥
            apply.UpsertVolumeMounts(&info.TemplateSpec.PodSets[psIdx].Containers[cIdx].VolumeMounts,
                *corev1ac.VolumeMount().
                    WithName(constants.MPISSHAuthVolumeName).
                    WithMountPath(*info.RuntimePolicy.MLPolicySource.MPI.SSHAuthMountPath),  // 默认 /root/.ssh
            )

            // Launcher 容器还需要 hostfile 和 OpenMPI 环境变量
            if ps.Name == constants.Launcher {
                apply.UpsertVolumeMounts(&info.TemplateSpec.PodSets[psIdx].Containers[cIdx].VolumeMounts,
                    *corev1ac.VolumeMount().
                        WithName(constants.MPIHostfileVolumeName).
                        WithMountPath(constants.MPIHostfileDir),  // /etc/mpi
                )

                // OpenMPI 环境变量
                apply.UpsertEnvVars(&info.TemplateSpec.PodSets[psIdx].Containers[cIdx].Env,
                    *corev1ac.EnvVar().
                        WithName(constants.OpenMPIEnvHostFileLocation).  // OMPI_MCA_orte_default_hostfile
                        WithValue("/etc/mpi/hostfile"),
                    *corev1ac.EnvVar().
                        WithName(constants.OpenMPIEnvKeepFQDNHostNames). // OMPI_MCA_orte_keep_fqdn_hostnames
                        WithValue("true"),
                    *corev1ac.EnvVar().
                        WithName(constants.OpenMPIEnvDefaultSlots).      // OMPI_MCA_orte_set_default_slots
                        WithValue(strconv.Itoa(int(*info.RuntimePolicy.MLPolicySource.MPI.NumProcPerNode))),
                    *corev1ac.EnvVar().
                        WithName(constants.OpenMPIEnvKeyRSHArgs).        // OMPI_MCA_plm_rsh_args
                        WithValue("-o ConnectionAttempts=10"),
                )
            }
        }
    }

    return nil
}
```

### Build - 生成 Secret 和 ConfigMap

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:248-267
func (m *MPI) Build(ctx context.Context, info *runtime.Info, trainJob *trainer.TrainJob) ([]apiruntime.ApplyConfiguration, error) {
    if info == nil || info.RuntimePolicy.MLPolicySource == nil || info.RuntimePolicy.MLPolicySource.MPI == nil {
        return nil, nil
    }

    var objects []apiruntime.ApplyConfiguration

    // SSH Secret 是不可变的，只在不存在时创建
    if err := m.client.Get(ctx, client.ObjectKey{Name: sshAuthSecretName(trainJob.Name), Namespace: trainJob.Namespace}, &corev1.Secret{}); err != nil {
        if client.IgnoreNotFound(err) != nil {
            return nil, err
        }
        secret, err := m.buildSSHAuthSecret(trainJob)
        if err != nil {
            return nil, fmt.Errorf("failed to build SSH Auth secret: %w", err)
        }
        objects = append(objects, secret)
    }

    // hostfile ConfigMap 每次都重新生成 (节点可能变化)
    return append(objects, m.buildHostFileConfigMap(info, trainJob)), nil
}
```

### SSH 密钥生成

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:269-300
func (m *MPI) buildSSHAuthSecret(trainJob *trainer.TrainJob) (*corev1ac.SecretApplyConfiguration, error) {
    // 生成 ECDSA P-521 密钥对
    privateKey, err := ecdsa.GenerateKey(elliptic.P521(), rand.Reader)
    if err != nil {
        return nil, err
    }

    // 编码私钥为 PEM 格式
    privateDER, err := x509.MarshalECPrivateKey(privateKey)
    if err != nil {
        return nil, err
    }
    privatePEM := pem.EncodeToMemory(&pem.Block{
        Type:  "EC PRIVATE KEY",
        Bytes: privateDER,
    })

    // 生成 SSH 公钥
    publicKey, err := ssh.NewPublicKey(&privateKey.PublicKey)
    if err != nil {
        return nil, err
    }

    return corev1ac.Secret(sshAuthSecretName(trainJob.Name), trainJob.Namespace).
        WithType(corev1.SecretTypeSSHAuth).
        WithData(map[string][]byte{
            corev1.SSHAuthPrivateKey:  privatePEM,           // ssh-privatekey
            constants.MPISSHPublicKey: ssh.MarshalAuthorizedKey(publicKey), // ssh-publickey
        }).
        WithImmutable(true).  // Secret 不可变，重启不会重新生成
        WithOwnerReferences(ownerRef), nil
}
```

### Hostfile 生成

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:306-332
func (m *MPI) buildHostFileConfigMap(info *runtime.Info, trainJob *trainer.TrainJob) *corev1ac.ConfigMapApplyConfiguration {
    var hostFile bytes.Buffer
    runLauncherAsNode := ptr.Deref(info.RuntimePolicy.MLPolicySource.MPI.RunLauncherAsNode, false)
    slots := ptr.Deref(info.RuntimePolicy.MLPolicySource.MPI.NumProcPerNode, 1)

    for _, ps := range info.TemplateSpec.PodSets {
        // 只处理 node PodSet (或当 runLauncherAsNode 时也包括 launcher)
        if !isNode(runLauncherAsNode, ps) {
            continue
        }

        switch *info.RuntimePolicy.MLPolicySource.MPI.MPIImplementation {
        case trainer.MPIImplementationOpenMPI:
            // 遍历 Pod 端点，生成 hostfile 条目
            for endpoint := range ps.Endpoints {
                hostFile.WriteString(fmt.Sprintf("%s slots=%d\n", endpoint, slots))
            }
        }
    }

    return corev1ac.ConfigMap(fmt.Sprintf("%s%s", trainJob.Name, constants.MPIHostfileConfigMapSuffix), trainJob.Namespace).
        WithData(map[string]string{
            constants.MPIHostfileName: hostFile.String(),
        }).
        WithOwnerReferences(ownerRef)
}
```

**生成的 hostfile 示例**:
```
my-job-node-0-0.my-job slots=8
my-job-node-0-1.my-job slots=8
my-job-node-0-2.my-job slots=8
my-job-node-0-3.my-job slots=8
```

### Watch 扩展

```go
// pkg/runtime/framework/plugins/mpi/mpi.go:227-246
func (m *MPI) ReconcilerBuilders() []runtime.ReconcilerBuilder {
    return []runtime.ReconcilerBuilder{
        // Watch ConfigMap 变化
        func(b *builder.Builder, cl client.Client, cache cache.Cache) *builder.Builder {
            return b.Watches(&corev1.ConfigMap{},
                handler.EnqueueRequestForOwner(m.client.Scheme(), m.client.RESTMapper(), &trainer.TrainJob{}, handler.OnlyControllerOwner()))
        },
        // Watch Secret 变化
        func(b *builder.Builder, cl client.Client, cache cache.Cache) *builder.Builder {
            return b.Watches(&corev1.Secret{},
                handler.EnqueueRequestForOwner(m.client.Scheme(), m.client.RESTMapper(), &trainer.TrainJob{}, handler.OnlyControllerOwner()))
        },
    }
}
```

## MPI 训练架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MPI Distributed Training                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Launcher Pod                                 │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  mpirun --hostfile /etc/mpi/hostfile                         │    │   │
│  │  │         -np 32                                                │    │   │
│  │  │         --bind-to none                                        │    │   │
│  │  │         python train.py                                       │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                      │   │
│  │  Volumes:                                                            │   │
│  │    - /root/.ssh/id_rsa (SSH Private Key)                            │   │
│  │    - /root/.ssh/id_rsa.pub (SSH Public Key)                         │   │
│  │    - /etc/mpi/hostfile (MPI Hostfile)                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │ SSH                                          │
│                              ▼                                              │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐               │
│  │   Node Pod 0    │ │   Node Pod 1    │ │   Node Pod N    │               │
│  │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │               │
│  │  │ sshd      │  │ │  │ sshd      │  │ │  │ sshd      │  │               │
│  │  │ train.py  │  │ │  │ train.py  │  │ │  │ train.py  │  │               │
│  │  │ (rank 0-7)│  │ │  │ (rank 8-15)│  │ │  │ (rank ..)│  │               │
│  │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │               │
│  │                 │ │                 │ │                 │               │
│  │  Volumes:       │ │  Volumes:       │ │  Volumes:       │               │
│  │  - /root/.ssh/* │ │  - /root/.ssh/* │ │  - /root/.ssh/* │               │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## OpenMPI 环境变量

| 环境变量 | 说明 | 值 |
|:---------|:-----|:---|
| `OMPI_MCA_orte_default_hostfile` | 默认 hostfile 路径 | `/etc/mpi/hostfile` |
| `OMPI_MCA_orte_keep_fqdn_hostnames` | 保持 FQDN 主机名 | `true` |
| `OMPI_MCA_orte_set_default_slots` | 默认 slots 数 | `8` (GPU 数) |
| `OMPI_MCA_plm_rsh_args` | RSH 参数 | `-o ConnectionAttempts=10` |

## runLauncherAsNode 模式

当 `runLauncherAsNode: true` 时，Launcher Pod 也参与训练：

```yaml
# TrainingRuntime 配置
spec:
  mlPolicy:
    mpi:
      runLauncherAsNode: true
      numProcPerNode: 8
```

此时：
- Launcher Pod 既运行 `mpirun` 也运行训练进程
- Node Pod 数量 = `numNodes - 1`
- Hostfile 包含 Launcher Pod 的端点

## 安全考虑

1. **SSH 密钥**: 使用 ECDSA P-521 椭圆曲线，提供高安全性
2. **Secret 不可变**: `WithImmutable(true)` 防止密钥被篡改
3. **最小权限**: 只有同一 TrainJob 的 Pod 才能访问 Secret
4. **OwnerReference**: TrainJob 删除时自动清理 Secret 和 ConfigMap
