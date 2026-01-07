# 🧪 模拟实战：Kubeflow Trainer 任务清单 (Exercises)

> **角色扮演说明**:
> 你现在是 Kubeflow Trainer 团队的 **Tech Lead**。
> 你的目标不是“读懂代码就结束”，而是把核心能力拆成一组可以交付的任务：**CRD 建模 → Reconcile 收敛 → Runtime/Plugin 扩展 → 可观测与运维**。
>
> 本清单默认你会为每个任务补齐：
> - 设计说明（1-2 页）
> - 单元测试/集成测试（envtest 或 kind）
> - 验收用的最小 YAML 示例

---

## Milestone 1: CRD 与心智模型 (The API)
**目标**: 先把“用户写什么、系统承诺什么”搞清楚，形成稳定的 API 与状态模型。
**参考**: `AI-Infrastructure/trainer/modules/controller.md`, `AI-Infrastructure/trainer/flows/trainjob_reconcile.md`

### 🟢 Task 1.1: TrainJob API 字段与不变量清单
*   **优先级**: P0
*   **需求**:
    1. 梳理 `TrainJob.spec` 的关键字段：`runtimeRef`, `trainer`, `initializer`, `podTemplateOverrides`, `suspend`, `managedBy`。
    2. 写出 10 条不变量（例如：`runtimeRef` 必填；`numNodes` 与 runtime 的默认值合并策略必须确定）。
    3. 画出状态机草图：`Pending → Running → Succeeded/Failed`，以及 `Suspended` 分支。
*   **验收标准**:
    - [ ] 输出一份 `TrainJob API Cheatsheet`（表格 + 字段含义 + 默认值/互斥关系）。
    - [ ] 输出一份 `TrainJob 状态机` 图（Mermaid）。

### 🟡 Task 1.2: TrainingRuntime / ClusterTrainingRuntime 合并规则
*   **优先级**: P0
*   **背景**: 运行时模板是“平台侧约束 + 默认值”的载体，TrainJob 是“用户输入”。
*   **需求**:
    1. 梳理 runtime（`TrainingRuntime`/`ClusterTrainingRuntime`）与 TrainJob 的合并优先级。
    2. 明确“平台强制项”与“用户可覆盖项”的边界（哪些字段允许 override）。
    3. 列出 3 个冲突示例（例如：GPU 资源、镜像、调度策略）。
*   **验收标准**:
    - [ ] 写出一份 `Merge Policy`（优先级表 + 冲突处理策略）。

---

## Milestone 2: Reconcile 收敛 (The Controller)
**目标**: 能把 TrainJob 收敛成一组 Kubernetes 资源，并正确更新状态。
**参考**: `AI-Infrastructure/trainer/flows/trainjob_reconcile.md`, `AI-Infrastructure/trainer/flows/training_execution.md`

### 🟢 Task 2.1: 最小可运行的 TrainJob Controller（Toy 实现）
*   **优先级**: P0
*   **需求**:
    1. 用 `controller-runtime` 搭一个最小 controller（单 CRD + 单 controller）。
    2. Reconcile 做三件事：读取 TrainJob → 生成子资源（先用 ConfigMap/Job 代替）→ 更新 Status。
    3. 处理删除：加 finalizer，清理子资源后移除 finalizer。
*   **验收标准**:
    - [ ] `envtest` 下能创建 TrainJob 并生成子资源。
    - [ ] 删除 TrainJob 时，子资源被清理，finalizer 被移除。

### 🟡 Task 2.2: SSA（Server-Side Apply）与字段所有权策略
*   **优先级**: P1
*   **背景**: Trainer 强依赖 SSA 来实现“声明式收敛 + 组件化生成”。
*   **需求**:
    1. 为 toy controller 的子资源改成 SSA 应用（ApplyConfiguration 或 Patch）。
    2. 定义字段所有权（Field Manager）与冲突策略（Force/不 Force）。
    3. 设计一次“用户手改子资源”的冲突场景，并说明系统应如何处理。
*   **验收标准**:
    - [ ] 给出一个冲突复现步骤与预期行为（文档）。

---

## Milestone 3: Runtime 层与插件化 (The Extensibility)
**目标**: 把“生成子资源”的逻辑从 controller 解耦出来，形成可插拔的 Runtime/Plugin。
**参考**: `AI-Infrastructure/trainer/modules/runtime.md`, `AI-Infrastructure/trainer/modules/plugin_framework.md`

### 🟢 Task 3.1: 抽象 Runtime 接口与 Info 上下文
*   **优先级**: P0
*   **需求**:
    1. 定义 `Runtime` 接口（`NewObjects`, `RuntimeInfo`, `TrainJobStatus`, `ValidateObjects`）。
    2. 设计 `Info` 结构体：包含 runtime 配置、合并后的 MLPolicy、派生出的训练拓扑（nodes/procs）。
    3. 给出 `Info` 的构建流程图（输入：TrainJob + Runtime；输出：Info）。
*   **验收标准**:
    - [ ] `Info` 构建流程可解释清楚“默认值/override/平台约束”的位置。

### 🟡 Task 3.2: 组件构建插件（JobSet / Secret / ConfigMap）
*   **优先级**: P1
*   **需求**:
    1. 以 `JobSet` 为例：定义 `ComponentBuilderPlugin`，输出一组 ApplyConfiguration。
    2. 写一个最小 `JobSetBuilder`：基于 `Info` 生成 1 个 replicated job（worker）。
    3. 设计“插件顺序”与“插件间数据传递”的策略（共享 Info vs 返回 patch）。
*   **验收标准**:
    - [ ] 用 2 个插件组合生成对象：`Secret` + `JobSet`。

---

## 进阶挑战 (Bonus)
**参考**: `AI-Infrastructure/trainer/tradeoffs.md`, `AI-Infrastructure/trainer/questions.md`

*   **Task 4.1**: 设计 TrainJob 的 Condition 集合（Ready/Running/Failed/Succeeded/Suspended），给出更新规则。
*   **Task 4.2**: 加入 coscheduling（PodGroup）策略：说明何时创建/更新，超时如何处理。
*   **Task 4.3**: 可观测性：列出关键指标（reconcile latency、apply error、jobset ready time）与日志结构。

