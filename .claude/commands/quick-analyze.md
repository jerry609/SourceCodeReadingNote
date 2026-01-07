# Quick Source Analysis Skill

你是一个源码分析助手。对给定项目进行快速分析，生成核心笔记框架。

## 输入

- **仓库路径**: $ARGUMENTS

## 快速分析步骤

### Step 1: 识别项目 (1-2分钟)

```bash
# 读取关键文件
README.md
go.mod / package.json / Cargo.toml / pyproject.toml
LICENSE
```

确定：
- 项目名称和类型
- 主要编程语言
- 分类目录（Storage/AI-Infrastructure/...）

### Step 2: 结构扫描 (2-3分钟)

```bash
# 扫描目录
ls -la
ls -la cmd/ pkg/ src/ lib/ internal/
```

识别：
- 入口点
- 核心模块
- 测试目录

### Step 3: 架构草图 (3-5分钟)

绘制高层架构图：
```
┌─────────────────┐
│     入口层       │
├─────────────────┤
│     业务层       │
├─────────────────┤
│     数据层       │
└─────────────────┘
```

### Step 4: 生成框架

创建目录结构：
```
[Category]/[ProjectName]/
├── README.md          # 填写项目概览
├── reading-guide.md   # 填写阅读指南
├── flows/             # 创建空目录
├── modules/
│   └── template.md    # 复制模板
├── questions.md       # 复制模板
├── highlights.md      # 复制模板
├── algorithms.md      # 复制模板
├── tradeoffs.md       # 复制模板
└── exercises/
    └── TASKS.md       # 创建任务清单
```

## 输出

生成两个核心文件：

1. **README.md** - 包含：
   - 项目概览
   - 架构图
   - 核心模块列表
   - 关键数据结构

2. **reading-guide.md** - 包含：
   - 3 个核心问题
   - 推荐阅读顺序
   - 锚点文件列表

## 模板位置

从 `00-TEMPLATE/` 复制模板文件。

## 执行

快速分析仓库：**$ARGUMENTS**
