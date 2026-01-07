# Update Index Skill

你是一个文档维护助手。更新 SourceCodeReadingNote 仓库的主 README.md。

## 输入

无需参数，自动扫描当前仓库。

## 执行步骤

### Step 1: 扫描目录

遍历所有分类目录：
- `Storage/` - 存储系统
- `AI-Infrastructure/` - AI 基础设施
- `AI-Models/` - AI 模型
- `Web-Applications/` - Web 应用
- `DevTools-IDE/` - 开发工具
- `Books/` - 技术书籍

### Step 2: 判断状态

| 状态 | 条件 |
|:-----|:-----|
| ✅ 完成 | README.md 内容完整，flows/ 和 modules/ 有实质内容 |
| 🔄 进行中 | 有 README.md 但部分内容为模板占位符 |
| 📝 待开始 | 仅有空目录或纯模板文件 |

### Step 3: 提取信息

从每个项目的 README.md 提取：
- **项目名称**: 目录名或 README 标题
- **项目类型**: 从"类型"字段提取
- **主要语言**: 从"核心语言"字段提取

### Step 4: 更新 README.md

更新根目录 `README.md` 的项目表格：

```markdown
## 项目列表

### 存储系统 (Storage)
| 项目 | 类型 | 语言 | 状态 |
|:-----|:-----|:-----|:-----|
| [JuiceFS](Storage/JuiceFS/) | 分布式 POSIX 文件系统 | Go | ✅ 完成 |

### AI 基础设施 (AI-Infrastructure)
| 项目 | 类型 | 语言 | 状态 |
|:-----|:-----|:-----|:-----|
| [Kubeflow Trainer](AI-Infrastructure/trainer/) | K8s 分布式训练 Operator | Go | ✅ 完成 |
```

## 输出

更新根目录的 `README.md`。

## 执行

扫描仓库并更新项目列表。
