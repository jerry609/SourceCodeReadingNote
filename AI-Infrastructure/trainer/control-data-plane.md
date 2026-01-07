# 控制面与数据面分离 (Control vs Data Plane)

> 大脑只做决策，四肢只做动作。

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      Control Plane (Trainer)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ API Server  │  │  Webhook    │  │  TrainJob Controller    │  │
│  │ (CRD 定义)  │  │  (验证)     │  │  (协调/收敛)            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                           │                                      │
│         ┌─────────────────┼─────────────────┐                   │
│         ▼                 ▼                 ▼                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Runtime   │  │   Plugin    │  │      JobSet API         │  │
│  │   Registry  │  │  Framework  │  │  (声明式配置下发)        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Data Plane    │ │   Data Plane    │ │   Data Plane    │
│   (Worker 0)    │ │   (Worker 1)    │ │   (Worker N)    │
│  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │
│  │ torchrun  │  │ │  │ torchrun  │  │ │  │ torchrun  │  │
│  │ 训练进程   │  │ │  │ 训练进程   │  │ │  │ 训练进程   │  │
│  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 控制面组件

| 组件 | 职责 | 状态管理 | 决策类型 | 源码位置 |
|:-----|:-----|:---------|:---------|:---------|
| TrainJob Controller | 协调 TrainJob 到 JobSet | 有状态 (etcd) | 协调 | `pkg/controller/` |
| Runtime Registry | 管理 Runtime 实现 | 无状态 | 查找 | `pkg/runtime/core/registry.go` |
| Plugin Framework | 执行扩展逻辑 | 无状态 | 策略 | `pkg/runtime/framework/` |
| Webhook | 验证和默认值 | 无状态 | 验证 | `pkg/webhook/` |

### 组件 1: TrainJob Controller

**职责**: 监听 TrainJob 变更，协调创建/更新 JobSet

**决策逻辑**:
```go
// pkg/controller/trainjob_controller.go:98-130
func (r *TrainJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取期望状态 (TrainJob)
    var trainJob trainer.TrainJob
    if err := r.client.Get(ctx, req.NamespacedName, &trainJob); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. 解析 Runtime，构建 Info
    runtimeSpec, err := runtimeObj.GetRuntimeSpec(ctx, &trainJob)
    info := r.framework.RunEnforceMLPolicyPlugins(info, trainJob, runtimeSpec)
    info = r.framework.RunEnforceCustomizationPlugins(info, trainJob, runtimeSpec)

    // 3. 构建 JobSet（期望状态 → 底层资源）
    objs, err := runtimeObj.BuildObjects(ctx, info, &trainJob, runtimeSpec)

    // 4. 应用到集群（声明式）
    for _, obj := range objs {
        r.client.Patch(ctx, obj, client.Apply, ...)
    }

    // 5. 同步状态
    return r.updateTrainJobStatus(ctx, &trainJob, objs)
}
```

**依赖**: 只依赖 Runtime 接口，不依赖具体 JobSet 实现细节

### 组件 2: Plugin Framework

**职责**: 提供可扩展的策略注入点

**决策逻辑**:
```go
// pkg/runtime/framework/core/framework.go
func (f *Framework) RunEnforceMLPolicyPlugins(info *runtime.Info, ...) *runtime.Info {
    for _, plugin := range f.enforceMLPolicyPlugins {
        info = plugin.EnforceMLPolicy(info, trainJob, runtimeSpec)
    }
    return info
}
```

**扩展机制**:
- MLPolicy 插件: Torch, MPI, DeepSpeed
- Customization 插件: PodGroupPolicy, JobSet

---

## 数据面组件

| 组件 | 职责 | 可替换性 | 环境依赖 | 源码位置 |
|:-----|:-----|:---------|:---------|:---------|
| JobSet | Pod 编排 | 低 | K8s | (外部依赖) |
| Worker Pod | 执行训练 | 高 | GPU/网络 | 用户镜像 |
| torchrun | 进程管理 | 高 | PyTorch | 用户环境 |

### 组件 1: Worker Pod

**职责**: 执行实际的训练计算

**接口契约**:
```yaml
# 控制面通过环境变量与数据面通信
env:
  - name: PET_NNODES
    value: "4"                    # 节点数
  - name: PET_NPROC_PER_NODE
    value: "8"                    # 每节点进程数
  - name: PET_MASTER_ADDR
    value: "job-worker-0"         # Master 地址
  - name: PET_MASTER_PORT
    value: "29500"                # Master 端口
```

**多实现支持**:
- PyTorch 训练: 使用 torchrun
- MPI 训练: 使用 mpirun + SSH
- 自定义框架: 实现相同环境变量契约

---

## 边界清晰度评估

### Checklist

- [x] 控制面不依赖执行面的细节实现（只依赖契约/接口）
  - Controller 只生成 JobSet spec，不关心 Pod 内部如何执行
- [x] 执行面可以异构与替换（不同环境/供应商/实现并存）
  - 支持 Torch/MPI/DeepSpeed 等多种训练框架
- [x] 状态只在控制面持久化，执行面是无状态或可重建的
  - TrainJob/JobSet 状态在 etcd，Worker Pod 是无状态的
- [x] 控制面和数据面可以独立扩缩容
  - Controller 副本数与 Worker 数量无关
- [ ] 网络分区时，数据面可以继续工作（优雅降级）
  - 部分支持：训练可继续，但新任务无法创建

### 违反分离的代码

| 位置 | 问题 | 影响 | 改进建议 |
|:-----|:-----|:-----|:---------|
| 无明显违反 | - | - | - |

---

## 通信模式

### 控制面 → 数据面

| 模式 | 描述 | 适用场景 |
|:-----|:-----|:---------|
| 声明式配置 | 通过 JobSet spec 下发配置 | 初始配置 |
| 环境变量 | 训练参数注入 Pod | 运行时参数 |
| ConfigMap | 配置文件挂载 | MPI hostfile |
| Secret | 敏感信息 | SSH 密钥 |

```
┌──────────────┐    JobSet Spec    ┌──────────────┐
│  Controller  │─────────────────▶│   JobSet     │
└──────────────┘                   │  Controller  │
                                   └──────────────┘
                                          │
                                   Pod Spec + Env
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │  Worker Pod  │
                                   └──────────────┘
```

### 数据面 → 控制面

| 模式 | 描述 | 适用场景 |
|:-----|:-----|:---------|
| Pod Status | K8s 原生状态上报 | 运行状态 |
| JobSet Status | 聚合训练状态 | 完成/失败 |
| Events | Kubernetes Events | 异常通知 |

```
┌──────────────┐   Watch JobSet    ┌──────────────┐
│  Controller  │◀─────────────────│   JobSet     │
└──────────────┘     Status        └──────────────┘
       │                                  ▲
       │                           Pod Status
       ▼                                  │
┌──────────────┐                   ┌──────────────┐
│  TrainJob    │                   │  Worker Pod  │
│   Status     │                   └──────────────┘
└──────────────┘
```

---

## 故障隔离

| 故障场景 | 控制面影响 | 数据面影响 | 恢复策略 |
|:---------|:-----------|:-----------|:---------|
| Controller 不可用 | 无法创建/更新任务 | 现有训练继续运行 | Controller 重启后自动恢复 |
| 单个 Worker 故障 | 检测并更新状态 | 训练可能失败 | JobSet 重启策略 |
| 网络分区 | 部分节点不可达 | 分布式训练失败 | 等待网络恢复 |
| etcd 不可用 | 无法持久化状态 | 现有训练继续 | etcd 恢复后重连 |

---

## 设计优点

1. **清晰的职责分离**
   - Controller 只负责"做什么"（声明 JobSet）
   - JobSet Controller 负责"怎么做"（调度 Pod）
   - Worker 负责"执行"（运行训练）

2. **可测试性**
   - 控制面可以单独测试，无需真实训练环境
   - 数据面可以 Mock（假 Worker）

3. **可观测性**
   - 控制面：Controller metrics, TrainJob status
   - 数据面：Pod metrics, 训练日志

4. **演进友好**
   - 新增训练框架只需实现插件
   - 不影响现有控制流程

---

## 参考对比

| 系统 | 控制面 | 数据面 |
|:-----|:-------|:-------|
| Kubeflow Trainer | Controller + Plugin Framework | JobSet + Worker Pods |
| Kubernetes | API Server + Controllers | kubelet + Container Runtime |
| Volcano | Scheduler + Controllers | Job Pods |
