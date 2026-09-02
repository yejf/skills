---
name: clip
display_name: 网页剪藏
display_name_en: Web Clipper
description: >-
  抓取网页内容，自动分类并提炼核心观点、金句，保存为结构化笔记。
  支持技术文章、教程、新闻、观点文、研究论文等各类网页。
  当用户粘贴 URL、说"收藏""剪藏""提炼这篇文章""帮我整理这个链接""clip this"时使用。
  也适用于用户想从网页内容中提取关键信息保存到知识库的场景。
description_zh: 抓取网页内容，自动分类并提炼核心观点、金句，保存为结构化笔记
description_en: Fetch web content, auto-classify and extract key insights, quotes, save as structured notes
category: writing
version: 2.3.0
author: yejf
---

# Web Clipper — 网页剪藏

将任意网页转化为结构化笔记，自动分类、提炼核心观点和金句。

## 触发条件

当用户执行以下操作时触发：
- 粘贴一个或多个 URL
- 说"收藏""剪藏""clip""save this"
- 说"提炼这篇文章""帮我整理这个链接"
- 说"总结一下这个网页""这篇文章说了什么"

## 前置依赖

执行前先阅读：
- `@references/fetch-strategy.md` — 三层抓取策略
- `@references/content-classifier.md` — 内容分类规则
- `@references/quality.md` — 质量标准

## 执行流程

### 第一步：获取内容

按 `@references/fetch-strategy.md` 中的策略获取网页内容。

**默认优先级**：

1. **WebFetch** — 最快，零配置，处理 80% 静态页面
2. **Firecrawl MCP** — 备选，需设置 `FIRECRAWL_API_KEY`
3. **Playwright MCP** — 最强，处理登录/付费墙/SPA

**Playwright 抓取流程**：
```
1. browser_navigate(url) — 导航到页面
2. browser_wait_for(text) — 等待内容加载
3. browser_snapshot() — 获取无障碍树快照
4. 提取正文内容和元数据
```

最终兜底：截取快照 + 保存原始 URL。

获取成功后，提取：
- 正文内容（Markdown 格式）
- 元数据：标题、作者、发布日期、来源 URL

### 第二步：内容分类

按 `@references/content-classifier.md` 中的规则判断文章类型：

| 类型 | 中文标签 | 判断特征 |
|------|----------|----------|
| `tech-article` | tech-article【技术文章】 | 含代码块、技术术语、API 名称、架构描述 |
| `tutorial` | tutorial【教程】 | 步骤式结构、"how to"、"guide"、"step" |
| `news` | news【新闻】 | 时间敏感、事件报道、发布声明、行业动态 |
| `opinion` | opinion【观点】 | 第一人称、论证结构、观点鲜明、立场表达 |
| `research` | research【研究论文】 | 引用密集、数据图表、学术风格、方法论描述 |
| `paper` | paper【学术论文】 | 作者机构、摘要、方法论、参考文献、DOI |
| `documentation` | documentation【官方文档】 | API 参考、配置说明、版本信息、代码示例 |
| `general` | general【通用】 | 以上都不匹配 |

**分类结果格式**：使用中文标签，如 `opinion【观点】`、`tutorial【教程】`。

分类结果决定后续使用的输出模板。

### 第三步：深度提炼

根据分类结果，读取对应的 reference 文件获取提炼规则：

- `tech-article` → `@references/tech-article.md`
- `tutorial` → `@references/tutorial.md`
- `news` → `@references/news.md`
- `opinion` → `@references/opinion.md`
- `research` → `@references/research.md`
- `paper` → `@references/paper.md`
- `documentation` → `@references/documentation.md`
- `general` → `@references/general.md`

每个 reference 文件包含：
1. 该类型的输出模板结构
2. 提炼规则（什么该提取、什么该忽略）
3. 金句提取规则
4. 质量检查清单

提炼时必须遵守 `@references/quality.md` 中的通用质量标准。

### 第四步：生成笔记

使用 `@templates/article-note.md` 模板生成最终笔记。

模板中的变量按以下方式填充：
- `{{title}}` — 文章标题
- `{{source}}` — 原始 URL
- `{{author}}` — 作者（未知则留空）
- `{{date_captured}}` — 今天的日期 YYYY-MM-DD
- `{{category}}` — 第二步的分类结果（使用中文标签格式，如 `opinion【观点】`）
- `{{tags}}` — 从内容中提取的标签（3-5 个）
- `{{reading_time}}` — 预估阅读时间（按 300 字/分钟）
- `{{word_count}}` — 正文字数

正文内容按分类模板的结构组织。

### 第五步：保存笔记

每次 `/clip` 执行后，会产生/更新以下文件：

| 文件 | 首次执行 | 后续执行 | 说明 |
|------|----------|----------|------|
| `clip/article/YYYY-MM-DD-xxx.md` | 新建 | 新建 | 提炼后的文章笔记 |
| `clip/collect.csv` | 新建（带 BOM + 表头） | 追加一行 | 结构化数据记录 |
| `clip/collect.html` | 新建 | 不动 | 可视化页面，自动从 CSV 加载 |

**首次执行检查**：如果 `clip/` 目录或 `collect.csv`、`collect.html` 不存在，先创建再继续。

#### 5.1 保存笔记文件

- 路径：`clip/article/YYYY-MM-DD-标题-kebab-case.md`
- 标题过长时截取前 30 个字符
- 文件名中的中文转为拼音或英文关键词

#### 5.2 更新 collect.csv（必须）

**首次执行**：创建 `clip/collect.csv`，写入表头 + 第一行数据：

```csv
id,date,title,author,source,category,word_count,url,file
1,YYYY-MM-DD,标题,作者,来源,分类【中文】,字数,URL,article/YYYY-MM-DD-xxx.md
```

**后续执行**：在末尾追加一行：

```
<新序号>,YYYY-MM-DD,标题,作者,来源,分类【中文】,字数,URL,article/YYYY-MM-DD-xxx.md
```

规则：
- 序号 = 当前最大序号 + 1
- CSV 编码：UTF-8 with BOM（兼容 Excel 打开）
- 无引号，逗号分隔
- `collect.html` 会自动从 CSV 加载数据，**不需要手动更新 HTML**

#### 5.3 创建 collect.html（仅首次）

首次执行时，创建 `clip/collect.html`。这是一个自包含的 HTML 页面，功能包括：
- 从 `collect.csv` 加载数据（XMLHttpRequest，兼容本地文件）
- 统计卡片（总文章数、总字数、分类数）
- 搜索框（按标题、作者、来源过滤）
- 分类筛选下拉菜单
- 表头点击排序
- 原文链接 + 笔记链接
- 响应式布局

完整代码参考：`@templates/collect.html.template`

#### 5.4 更新知识库（可选）

- 如果提炼中发现与 `knowledge/topics/` 中已有主题相关，添加关联
- 新术语创建 `knowledge/glossary/` 卡片

#### 5.5 向用户报告

- 告知笔记保存位置
- 告知 collect.csv 已更新（浏览器打开 collect.html 可查看汇总）
- 展示一句话总结和金句（1-2 句）
- 询问是否需要进一步操作（如写公众号文章）

## 存储结构

```
clip/
├── collect.csv         # 数据源（每次 /clip 追加一行）
├── collect.html        # 展示层（浏览器打开，自动读取 CSV）
├── article/            # 文章笔记（每次 /clip 新建一个）
│   ├── 2026-09-02-article-1.md
│   ├── 2026-09-02-article-2.md
│   └── ...
└── (skill 文件在 .claude/skills/clip/)
```

**数据流**：`/clip` 执行 → 新建笔记 + 追加 CSV → 浏览器打开 HTML 自动展示

**文件职责**：
- `collect.csv` — 唯一数据源，存储所有收录文章的元信息（序号、日期、标题、作者、来源、分类、字数、URL、文件路径）
- `collect.html` — 纯展示层，通过 JavaScript 读取 CSV 渲染表格，支持搜索、筛选、排序，**不存储数据，无需更新**
- `article/*.md` — 文章笔记正文，由 skill 自动生成

## 错误处理

| 情况 | 处理方式 |
|------|----------|
| URL 无法访问 | 提示用户检查 URL，询问是否粘贴正文 |
| 内容为空 | 提示页面可能需要登录，建议用 Playwright 重试 |
| 分类不确定 | 默认使用 `general【通用】` 类型 |
| 内容过短（<200字） | 提示内容太少，询问是否仍要保存 |
| 内容过长（>50000字） | 只提炼前 50000 字，标注"部分内容" |

## 使用示例

```
用户：收藏这个链接 https://example.com/article
AI：[执行抓取→分类→提炼→保存]
    笔记已保存到 clip/article/2026-09-02-article-title.md
    汇总表格已更新（第 4 条）
    一句话总结：...
    金句：「...」
```

```
用户：帮我整理这几篇文章
    https://example.com/article1
    https://example.com/article2
AI：[逐一处理，批量保存，统一更新汇总表格]
```
