# 控制面与数据面分离 (Control vs Data Plane)

> 大脑只做决策，四肢只做动作。

在 Kubeflow Trainer 中：
- **控制面（Trainer）**：只负责把 TrainJob 转换成一组声明式 K8s 资源，并把底层状态聚合回 TrainJob
- **数据面（训练任务）**：由 JobSet/Job/Pod 执行实际训练进程（torchrun/torchtune/mpirun…）

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      Control Plane (Trainer)                     │
│  ┌─────────────┐  ┌──────────────────┐  ┌─────────────────────┐ │
│  │ API Server  │  │ ValidatingWebhook │  │ TrainJob Controller  │ │
│  │ (CRDs)      │  │ (ValidateObjects) │  │ (SSA Apply + Status) │ │
│  └─────────────┘  └──────────────────┘  └─────────────────────┘ │
│                 ┌──────────────────────────────────────────────┐ │
│                 │ Runtime + Plugin Framework (policy/builders) │ │
│                 └──────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                            │   声明式对象（JobSet/PodGroup/…）
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Data Plane (K8s)                         │
│  ┌───────────────────┐   ┌───────────────────┐                  │
│  │ JobSet Controller  │→  │ Job / Pod         │→ 训练进程/日志    │
│  └───────────────────┘   └───────────────────┘                  │
│        ▲                     │                                   │
│        └──── Status/Events ───┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 控制面组件与职责

| 组件 | 职责 | 代码位置 |
|:--|:--|:--|
| TrainJob Controller | 选择 runtime、Apply 子资源、回写 status | `pkg/controller/trainjob_controller.go` |
| Runtime | 把 (Cluster)TrainingRuntime 模板变成 `runtime.Info` + 对象列表 | `pkg/runtime/core/*` |
| Plugin Framework | 按阶段执行策略/构建/校验/扩展 watch | `pkg/runtime/framework/*` |
| Webhook | admission 阶段委托 runtime 校验 | `pkg/webhooks/trainjob_webhook.go` |
| Runtime Controllers | 给 (Cluster)TrainingRuntime 加/移除 finalizer（防误删） | `pkg/controller/*trainingruntime_controller.go` |

控制面不关心“Pod 里怎么跑训练”，只关心“把应该跑的东西声明出来”。

---

## 数据面组件与职责

| 组件 | 职责 | 可替换性 |
|:--|:--|:--|
| JobSet | 把多个 replicated jobs 组织成一个整体 workload | 低（外部依赖） |
| Job/Pod | 执行容器，承载训练/initializer | 中（由模板决定） |
| 训练进程 | torchrun/torchtune/mpirun 等 | 高（用户镜像/脚本决定） |

---

## 控制面 → 数据面：主要“通信通道”

Trainer 通过“声明式配置 + 少量约定”把控制意图传给数据面：

1) **Pod template（最核心）**
- runtime 模板定义基础 PodSpec
- TrainJob 通过 `.spec.trainer/*` 与 `.spec.podTemplateOverrides` 做覆盖

2) **环境变量契约**
- Torch：注入 `PET_*`（节点数、rank、master addr/port 等），由 torchrun/torchtune 读取
- MPI：注入 OpenMPI 相关 env（hostfile、slots、rsh 参数等）

3) **ConfigMap / Secret**
- MPI：SSH key（Secret，不可变）与 hostfile（ConfigMap）

4) **Volumes**
- initializer/训练共享数据目录（dataset/model/checkpoint）通常靠 PVC/emptyDir 等卷挂载完成

---

## 数据面 → 控制面：状态与可观测性

Trainer 主要通过 JobSet 聚合状态：
- JobSet controller 写 `JobSet.Status`
- JobSet 插件读 JobSet 并映射到 TrainJob 条件（Complete/Failed 等）
- TrainJob controller Patch `TrainJob.Status`

异常可观测性：
- 子资源创建失败：TrainJob controller 记录 Event（`TrainJobResourcesCreationFailed`）
- 训练日志：由 Pod 日志系统承载，不回传到 Trainer

---

## 边界清晰度 Checklist

- [x] 控制面不依赖训练进程细节（只依赖 JobSet/PodSpec/env 契约）
- [x] 数据面可替换（镜像/命令/框架由用户或 runtime 模板决定）
- [x] 状态聚合有单一来源（JobSet → TrainJob）
- [x] 控制面与数据面可独立扩缩容（controller 副本数与训练规模解耦）

