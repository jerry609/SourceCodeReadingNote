# 启动流程 (Startup)

本文记录 `trainer-controller-manager` 的启动链路：从读取配置到启动 Controller/Webhook。

## 入口与关键文件

- 入口：`cmd/trainer-controller-manager/main.go`
- 配置加载：`pkg/config/config.go`
- Runtime 注册与初始化：`pkg/runtime/core/core.go`, `pkg/runtime/core/registry.go`
- Controller 注册：`pkg/controller/setup.go`
- Webhook 注册：`pkg/webhooks/setup.go`
- 证书管理：`pkg/util/cert/*`（在 `main.go` 中调用）

## 启动时序

```
main()
  ├─ init(): AddToScheme(...)
  ├─ config.Load(...)                      // 读取/校验配置，组装 ctrl.Options
  ├─ ctrl.NewManager(cfg, options)         // 创建 controller-runtime Manager
  ├─ cert.ManageCerts(...)                 // (可选) webhook 证书轮换，写入 Secret
  ├─ setupProbeEndpoints(...)              // healthz/readyz（readyz 等待证书+webhook server）
  ├─ runtimecore.New(ctx, client, indexer) // 初始化 runtimes map（含索引器、插件框架）
  ├─ go setupControllers(mgr, runtimes)    // 等待证书 ready 后注册 Controllers/Webhooks
  └─ mgr.Start(ctx)                        // 启动 cache、controllers、webhook server、metrics
```

## 关键点拆解

### 1) 配置加载：`config.Load`

`pkg/config/config.go:Load` 支持两种来源：
- 未指定 `--config`：对 `Configuration` 应用默认值（scheme 默认化）
- 指定 `--config`：严格解码配置文件并校验

最终影响 `ctrl.Options` 的典型字段：
- Metrics server：地址/是否 TLS、以及默认禁用 HTTP/2（避免相关 CVE）
- Webhook server：host/port/TLS
- Health probe：`HealthProbeBindAddress`
- Leader election：租约/锁/namespace/ID
- Controller 并发：`GroupKindConcurrency`

### 2) Runtime 初始化：`runtimecore.New`

`pkg/runtime/core/core.go:New` 构建 `map[string]runtime.Runtime`：
- registry 来自 `pkg/runtime/core/registry.go:NewRuntimeRegistry`
- 支持两类 runtime：
  - `TrainingRuntime`（namespaced）
  - `ClusterTrainingRuntime`（cluster-scoped）
- `ClusterTrainingRuntime` 依赖 `TrainingRuntime` 先初始化（共享 `trainingRuntimeFactory`），因此 registry 显式声明依赖关系并在 `New()` 中保证顺序。

同时，`TrainingRuntime` 初始化时会设置多个索引器（`IndexField`），用于后续 WatchExtension/快速查找（见 `pkg/runtime/core/trainingruntime.go:NewTrainingRuntime`）。

### 3) Controller/Webhook 延迟注册

`main.go` 里通过 `certsReady` gate：
- healthz：直接 ping
- readyz：证书 ready 且 webhook server 监听成功后才返回 ready
- controller/webhook 注册放到 goroutine 里，并在 `setupControllers()` 内等待 `<-certsReady` 后执行：
  - `pkg/controller/setup.go:SetupControllers`
  - `pkg/webhooks/setup.go:Setup`

这样可以避免“控制器已启动但 webhook 证书未就绪”导致早期请求被拒绝的窗口期扩大。

