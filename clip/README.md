# Clip Skill

[English](#english) | [中文](#中文)

---

## English

### Overview

**Clip** is a Claude Code skill for collecting, organizing, and managing web articles, tutorials, news, opinions, and other online content. It automatically categorizes, summarizes, and saves content to your local knowledge base in structured Markdown format.

### Features

- **8 Content Categories**: Automatically classify content into Tutorial, Tech Article, Opinion, News, Research, General, Paper, Documentation
- **Quality Control**: L1/L2 filtering standards to ensure high-quality content collection
- **Smart Fetching**: Prioritize primary sources, avoid plagiarism sites
- **Structured Notes**: Automatically generate standardized Markdown notes with metadata
- **Quality Scoring**: 0-100 quality scoring system for collected content
- **Duplicate Detection**: URL and title similarity checking to avoid duplicates
- **Batch Support**: Support collecting entire series or multi-part articles

### Installation

1. Clone this repository to your skills directory:
   ```bash
   git clone <repository-url> ~/.claude/skills/
   ```

2. The skill will be automatically available in Claude Code

### Usage

Simply provide a URL to Claude Code and use the `/clip` command or natural language:

```
/clip https://example.com/article
```

Or in natural language:
```
Clip this article: https://example.com/article
```

### Supported Content Types

| Category | Description |
|----------|-------------|
| `tutorial` | Step-by-step guides with code examples |
| `tech-article` | Technical deep dives, architecture, performance |
| `opinion` | Thought leadership, predictions, debates |
| `news` | Product launches, version releases, industry events |
| `research` | Data, surveys, benchmarks, experiments |
| `paper` | Academic papers, conference papers, journal articles |
| `documentation` | Official docs, API references, configuration guides |
| `general` | Mixed content, career, management |

### Output Structure

Collected notes are saved in the following structure:

```
notes/
├── {category}/
│   ├── {YYYY-MM}-{slug}.md
│   └── ...
```

Each note includes:
- YAML frontmatter with metadata
- One-sentence summary
- TL;DR (for articles > 3000 words)
- Core content points
- Notable quotes
- Tags
- Action items (optional)

### Quality Scoring

Content is scored 0-100 based on:
- Source authority
- Content depth
- Practical value
- Writing quality
- Information density

### Configuration

The skill uses the following default paths:
- Notes directory: `notes/` (relative to current directory)
- Index file: `notes/INDEX.md`

You can customize these in your `CLAUDE.md`:
```markdown
clip_notes_dir: my-clip-notes/
```

### References

Detailed classification guidelines are available in the `references/` directory:
- `tutorial.md` - Tutorial classification rules
- `tech-article.md` - Tech article classification rules
- `opinion.md` - Opinion piece classification rules
- `news.md` - News classification rules
- `research.md` - Research content classification rules
- `paper.md` - Academic paper classification rules
- `documentation.md` - Documentation classification rules
- `general.md` - General content classification rules
- `quality.md` - Quality control standards
- `fetch-strategy.md` - Content fetching strategy
- `content-classifier.md` - Content classification system

### Template

A standardized article note template is available in `templates/article-note.md`.

### License

MIT

---

## 中文

### 概述

**Clip** 是一个 Claude Code 技能，用于收集、整理和管理网络文章、教程、新闻、观点等在线内容。它会自动分类、摘要并保存内容到本地知识库，采用结构化的 Markdown 格式。

### 功能特性

- **8 种内容分类**：自动将内容分为教程、技术文章、观点、新闻、研究、通用、论文、文档
- **质量控制**：L1/L2 过滤标准，确保高质量内容收集
- **智能抓取**：优先使用原始来源，避免采集站
- **结构化笔记**：自动生成标准化的 Markdown 笔记，包含元数据
- **质量评分**：0-100 分的质量评分系统
- **去重检测**：URL 和标题相似度检查，避免重复收集
- **批量支持**：支持收集系列文章或多部分内容

### 安装

1. 克隆此仓库到你的技能目录：
   ```bash
   git clone <repository-url> ~/.claude/skills/
   ```

2. 技能将在 Claude Code 中自动可用

### 使用方法

只需提供 URL 给 Claude Code 并使用 `/clip` 命令或自然语言：

```
/clip https://example.com/article
```

或使用自然语言：
```
帮我收集这篇文章：https://example.com/article
```

### 支持的内容类型

| 分类 | 描述 |
|------|------|
| `tutorial` | 包含代码示例的分步指南 |
| `tech-article` | 技术深度文章、架构、性能 |
| `opinion` | 观点文章、预测、辩论 |
| `news` | 产品发布、版本更新、行业事件 |
| `research` | 数据、调查、基准测试、实验 |
| `paper` | 学术论文、会议论文、期刊文章 |
| `documentation` | 官方文档、API 参考、配置指南 |
| `general` | 混合内容、职业、管理 |

### 输出结构

收集的笔记按以下结构保存：

```
notes/
├── {category}/
│   ├── {YYYY-MM}-{slug}.md
│   └── ...
```

每个笔记包含：
- YAML frontmatter 元数据
- 一句话总结
- TL;DR（文章超过 3000 字时）
- 核心内容要点
- 金句摘录
- 标签
- 行动启发（可选）

### 质量评分

内容按 0-100 分评分，基于：
- 来源权威性
- 内容深度
- 实用价值
- 写作质量
- 信息密度

### 配置

技能使用以下默认路径：
- 笔记目录：`notes/`（相对于当前目录）
- 索引文件：`notes/INDEX.md`

你可以在 `CLAUDE.md` 中自定义：
```markdown
clip_notes_dir: my-clip-notes/
```

### 参考文档

详细的分类指南位于 `references/` 目录：
- `tutorial.md` - 教程分类规则
- `tech-article.md` - 技术文章分类规则
- `opinion.md` - 观点文章分类规则
- `news.md` - 新闻分类规则
- `research.md` - 研究内容分类规则
- `paper.md` - 学术论文分类规则
- `documentation.md` - 官方文档分类规则
- `general.md` - 通用内容分类规则
- `quality.md` - 质量控制标准
- `fetch-strategy.md` - 内容抓取策略
- `content-classifier.md` - 内容分类系统

### 模板

标准化的文章笔记模板位于 `templates/article-note.md`。

### 许可证

MIT
