# 快速建立心智模型 (Reading Guide)

这份笔记的目标不是“把每行代码读完”，而是用最短路径建立**可用的心智模型**：你知道它在 Kubernetes 上“看什么、算什么、改什么、怎么收敛”。

---

## 0. 项目身份（30 秒）

一句话：**Kubeflow Trainer 是一个 Kubernetes Operator**，把用户的 `TrainJob` 转换成一组底层 K8s 资源（主要是 JobSet），并把状态聚合回 `TrainJob.Status`。

你只需要先确认三件事：
- 它的用户是谁：平台团队（提供 runtime 模板）+ 训练提交者（提交 TrainJob）
- 它的输入是什么：TrainJob + (Cluster)TrainingRuntime
- 它的输出是什么：JobSet/PodGroup/Secret/ConfigMap…（由插件决定）

入口文件（控制面启动）：`cmd/trainer-controller-manager/main.go`

---

## 1. 架构骨架（5 分钟）

建议先扫目录，回答 Operator 三连问：
1) **Watch 什么资源？**
2) **创建/管理什么资源？**
3) **状态机怎么实现？**

对应到这个项目的关键目录：
- `pkg/apis/`：CRD 类型定义（TrainJob、TrainingRuntime、ClusterTrainingRuntime…）
- `pkg/controller/`：多个 controller（TrainJob 主控制器 + runtime 防误删控制器）
- `pkg/runtime/`：运行时抽象（把 runtime 模板 + TrainJob 合成最终对象）
- `pkg/runtime/framework/`：插件框架（按阶段执行策略/构建/校验/扩展 watch）
- `manifests/` / `charts/`：部署与 CRD 生成物

---

## 2. 追数据流（核心：10 分钟）

把流程当成“函数管道”来读：

```
kubectl apply TrainJob
  ├─ admission: TrainJobWebhook.ValidateCreate/Update
  │    └─ runtime.ValidateObjects(...) -> 插件校验
  └─ controller: TrainJobReconciler.Reconcile
       ├─ runtime.NewObjects(...) -> 期望的 ApplyConfiguration 列表
       ├─ SSA Apply 子资源（JobSet/PodGroup/Secret/ConfigMap…）
       └─ runtime.TrainJobStatus(...) -> 回写 TrainJob.Status
```

相关走读文件（按“数据流顺序”）：
- `AI-Infrastructure/trainer/flows/startup.md`
- `AI-Infrastructure/trainer/flows/trainjob_reconcile.md`
- `AI-Infrastructure/trainer/flows/training_execution.md`

---

## 3. 找“锚点文件”（80/20）

理解这几个文件，你基本就掌握了项目 80% 的工作方式：

**CRD（输入/输出契约）**
- `pkg/apis/trainer/v1alpha1/trainjob_types.go`
- `pkg/apis/trainer/v1alpha1/trainingruntime_types.go`

**主闭环（Reconcile）**
- `pkg/controller/trainjob_controller.go`

**Runtime 抽象（把模板变成对象）**
- `pkg/runtime/interface.go`
- `pkg/runtime/core/trainingruntime.go`
- `pkg/runtime/core/clustertrainingruntime.go`
- `pkg/runtime/runtime.go`（`runtime.Info` 数据结构）

**插件框架（扩展点）**
- `pkg/runtime/framework/interface.go`
- `pkg/runtime/framework/core/framework.go`
- `pkg/runtime/framework/plugins/registry.go`

（对应笔记：`AI-Infrastructure/trainer/modules/controller.md`、`AI-Infrastructure/trainer/modules/runtime.md`、`AI-Infrastructure/trainer/modules/plugin_framework.md`、`AI-Infrastructure/trainer/extension-points.md`）

---

## 4. CRD 总览：多少个？哪些需要状态机？

按业务 CRD 来看（忽略配置类 CRD）：

| CRD | 作用域 | 是否有状态机 | 原因 |
|:--|:--|:--|:--|
| `TrainJob` | Namespaced | ✅ | 工作负载，需要表达生命周期与底层状态 |
| `TrainingRuntime` | Namespaced | ❌ | 模板/策略载体，主要由 finalizer 保护防误删 |
| `ClusterTrainingRuntime` | Cluster | ❌ | 模板/策略载体，主要由 finalizer 保护防误删 |

“状态机”在 Trainer 里的实现方式是 Kubernetes 典型做法：
- 状态存储：`status.conditions[]`（`Suspended/Complete/Failed` 等）
- 状态来源：
  - `Suspended`：由 TrainJob controller 根据 `spec.suspend` 直接设置
  - `Complete/Failed`：由 JobSet 插件从 `JobSet.Status.Conditions` 映射而来

细节见：`AI-Infrastructure/trainer/reconcile-loops.md`、`AI-Infrastructure/trainer/questions.md`

---

## 5. Operator 为什么偏爱“声明式状态机”？

一句话：分布式环境里**事件可能丢/乱序/重放**，而 reconcile 的本质是“每次重新计算期望并收敛到正确状态”。

你可以把它理解为：
- 显式 FSM：事件驱动，状态按转换表推进（适合订单/审批/协议）
- 声明式 reconcile：状态是“推导出来的”，靠幂等纠偏自愈（适合 K8s 控制器）

如果你想把“状态机类型”当作通用工具箱：

| 类型 | 典型场景 | 关键词 |
|:--|:--|:--|
| 显式 FSM | 业务流程/协议栈 | 严格路径、可测试 |
| 声明式 reconcile | K8s Operator/IaC | 幂等、自愈、最终一致 |
| 层次状态机（Statecharts） | 复杂 UI/嵌入式控制 | 子状态复用、避免状态爆炸 |
| 事件驱动/Actor | 高并发消息处理 | 异步、解耦、邮箱模型 |
| 规则驱动（推导式） | 聚合状态 | 规则优先级、无持久状态 |

---

## 6. Runtime 层怎么读（最短路径）

把 Runtime 想成一个“编排引擎”：
- 输入：TrainJob + (Cluster)TrainingRuntime（模板 + 策略）
- 中间态：`runtime.Info`（PodSets + Policy + Scheduler + metadata）
- 输出：一组 `ApplyConfiguration`（最终要 SSA Apply 的对象）

核心管线（阶段固定）：
1) `EnforceMLPolicy`（PlainML/Torch/MPI…）
2) `EnforcePodGroupPolicy`（CoScheduling/Volcano…）
3) `PodNetwork`（JobSet：生成 endpoints）
4) `ComponentBuilder`（JobSet/MPI/PodGroup…）

对应笔记入口：
- `AI-Infrastructure/trainer/modules/runtime.md`
- `AI-Infrastructure/trainer/extension-points.md`

---

## 7. 最有效的验证方式（10 分钟）

读完“数据流 + 锚点文件”后，做一个最小闭环验证：
1) 找一个 `examples/` 里的 TrainJob
2) 跟一遍：webhook 校验 → reconcile 生成对象 → Apply → JobSet 状态映射
3) 改一行（例如：env/numNodes 的注入），看 JobSet spec 是否变化

这样你会非常快地把“文字理解”变成“可操作理解”。

