# [Project Name] 源码分析笔记

## 1. 项目概览 (Overview)
*   **类型**: (e.g., 分布式文件系统, 强化学习框架)
*   **核心语言**: (e.g., Go, Python, TypeScript)
*   **一句话描述**: 解决什么核心问题？
*   **阅读目标**: (e.g., 理解元数据管理，用 Rust 重写)

## 2. 核心架构 (Architecture)
*(在此处绘制或描述系统的高层架构图)*

*   **组件 A**: 职责描述
*   **组件 B**: 职责描述

## 3. 关键路径 (Critical Paths)
*   [启动流程](flows/startup.md)
*   [核心业务流程 1 (e.g. Write)](flows/write_path.md)
*   [核心业务流程 2 (e.g. Read)](flows/read_path.md)

## 4. 模块分析 (Modules)
*   [模块 A](modules/module_a.md)
*   [模块 B](modules/module_b.md)

## 5. 重点类/结构体 (Key Structures)
*   `StructName`: 作用及生命周期

## 6. 惊艳之处 (Highlights)
*   [惊艳之处](highlights.md): 令人印象深刻的设计、优雅的实现、巧妙的技巧

## 7. 关键算法 (Key Algorithms)
*   [关键算法](algorithms.md): 核心算法原理、复杂度、实现细节

## 8. 权衡取舍 (Tradeoffs)
*   [权衡取舍](tradeoffs.md): 设计决策的利弊分析

## 9. 疑问与解答 (Q&A)
*   [问题列表](questions.md)

## 10. 系统设计哲学 (System Design Philosophy)
*   [不变量分析](invariants.md): 识别系统必须维护的核心不变量
*   [控制面与数据面分离](control-data-plane.md): 决策与执行的边界划分
*   [闭环设计](reconcile-loops.md): Reconcile Loop 模式分析
*   [扩展点设计](extension-points.md): 可插拔架构与插件机制
*   [演进策略](evolution-strategy.md): 版本管理、特性开关、迁移策略
*   [反模式识别](anti-patterns.md): 识别并避免常见反模式
*   [真相之源分析](sot-analysis.md): 数据权威性与一致性保证

## 11. 思考与重构 (Reflections)
*   如果用 Rust 重写，这里怎么设计？
*   这个设计模式的优缺点是什么？
