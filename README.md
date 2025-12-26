# SourceCodeReadingNote

> 优秀开源项目源码阅读笔记，记录架构设计、核心算法、惊艳实现与权衡取舍。

## 项目列表

### 存储系统 (Storage)
| 项目 | 类型 | 语言 | 状态 |
|:-----|:-----|:-----|:-----|
| [JuiceFS](Storage/JuiceFS/) | 分布式 POSIX 文件系统 | Go | ✅ 进行中 |

### AI 基础设施 (AI-Infrastructure)
*待补充...*

### AI 模型 (AI-Models)
*待补充...*

### Web 应用 (Web-Applications)
*待补充...*

### 开发工具 (DevTools-IDE)
*待补充...*

## 笔记结构

每个项目的笔记遵循统一模板 ([00-TEMPLATE](00-TEMPLATE/))：

```
Project/
├── README.md          # 项目概览、架构图、核心结构体
├── flows/             # 关键路径分析 (Lookup, Read, Write...)
├── modules/           # 模块详解 (含源码位置引用)
├── questions.md       # 疑问与解答
├── highlights.md      # 惊艳之处 - 值得学习的设计
├── algorithms.md      # 关键算法 - 原理与复杂度分析
├── tradeoffs.md       # 权衡取舍 - 设计决策分析
└── exercises/         # 实战练习 (如 Rust 重写任务)
```

## 笔记特色

- **源码位置引用**: 所有分析都标注具体文件和行号 (如 `pkg/meta/interface.go:116`)
- **Mermaid 流程图**: 关键流程用图表展示
- **代码片段**: 保留核心实现代码，便于理解
- **Rust 重写思考**: 探索如何用 Rust 实现相同功能
- **惊艳之处**: 记录令人印象深刻的设计，可在其他项目借鉴
- **算法详解**: 包含复杂度分析和伪代码
- **权衡分析**: 理解每个设计决策的利弊

## 如何使用

1. **快速了解项目**: 阅读项目的 `README.md`
2. **深入某个模块**: 查看 `modules/` 下对应文件
3. **理解核心流程**: 阅读 `flows/` 下的路径分析
4. **学习设计技巧**: 阅读 `highlights.md` 和 `algorithms.md`
5. **理解设计决策**: 阅读 `tradeoffs.md`

## License

MIT
