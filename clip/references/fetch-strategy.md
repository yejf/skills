# 三层抓取策略

## 概述

为确保 100% 抓取成功率，采用三层 fallback 策略。每层失败后自动升级到下一层。

```
Layer 1: WebFetch        → 最快，零配置，处理 80% 静态页面
Layer 2: Firecrawl MCP   → 需 API Key，处理 JS 渲染页面
Layer 3: Playwright MCP  → 最强，处理登录/付费墙/SPA
```

## 策略选择建议

根据目标网站类型选择策略：

| 网站类型 | 推荐策略 | 原因 |
|----------|----------|------|
| 静态博客、文档站 | WebFetch | 速度快，成功率高 |
| JS 渲染的 SPA | Firecrawl MCP | 云端渲染，输出干净 |
| 需要登录/付费墙 | Playwright MCP | 完整浏览器能力 |
| 微信公众号等国内站点 | Playwright MCP | 网络访问限制 |

**配置 Firecrawl**：
```bash
export FIRECRAWL_API_KEY=your_key
```

## Layer 1: WebFetch（首选）

**适用场景**：静态页面、博客、文档站、新闻站

**调用方式**：
```
WebFetch(url, prompt="提取文章正文内容，保留格式，忽略导航、广告、页脚等非正文内容")
```

**优势**：
- 零配置，无需 API Key
- 速度最快（直接 HTTP 请求）
- 输出干净的 Markdown

**局限**：
- 不执行 JavaScript（SPA 页面拿不到内容）
- 不处理登录/付费墙
- 某些反爬站点会拦截

**失败判断**：
- 返回内容 < 200 字
- 返回内容包含 "Access Denied"、"403"、"Please enable JavaScript" 等错误标识
- 返回内容主要是导航/菜单而非正文

**失败后**：升级到 Layer 2

## Layer 2: Firecrawl MCP（备选）

**适用场景**：JS 渲染页面、需要等待加载的 SPA、反爬严格的站点

**前置条件**：需要 `FIRECRAWL_API_KEY` 环境变量

**调用方式**：
```bash
# 使用 firecrawl-scrape skill
npx firecrawl scrape <url> --format markdown
```

**优势**：
- 云端渲染 JavaScript
- 输出格式干净，专为 LLM 优化
- 支持批量抓取

**局限**：
- 需要 API Key（付费服务）
- 无法处理登录/付费墙
- API 调用有延迟

**失败判断**：
- API 返回错误
- 返回内容为空或过短

**失败后**：升级到 Layer 3

## Layer 3: Playwright MCP（最后手段）

**适用场景**：需要登录的页面、付费墙、复杂 SPA、需要交互的页面

**调用方式**：
```
1. browser_navigate(url) — 导航到页面
2. browser_wait_for(text) — 等待内容加载
3. browser_snapshot() — 获取无障碍树快照
4. 如果需要登录：
   - browser_fill_form() — 填写登录表单
   - browser_click() — 点击按钮
   - browser_wait_for() — 等待登录完成
   - 重新获取 snapshot
```

**优势**：
- 完整浏览器能力
- 可处理登录和交互
- 支持所有现代网页

**局限**：
- 启动浏览器较慢
- 消耗本地资源
- snapshot 输出可能很长

**失败判断**：
- 页面加载超时
- 快照内容为空
- 遇到无法绕过的验证

**最终兜底**：如果三层都失败
1. 截取页面快照（browser_take_screenshot）
2. 保存原始 URL 到 `inbox/`
3. 向用户报告失败原因，建议手动处理

## 输出格式

无论哪一层成功，输出统一格式：

```json
{
  "success": true,
  "layer": "webfetch|firecrawl|playwright",
  "content": "markdown 格式的正文内容",
  "metadata": {
    "title": "文章标题",
    "author": "作者",
    "url": "原始 URL",
    "date": "发布日期",
    "word_count": 1234
  }
}
```

## 特殊情况处理

### 多页面文章
如果文章分多页（如"下一页"按钮）：
1. 先用 Playwright 抓取当前页
2. 检测是否有"下一页"链接
3. 递归抓取所有页面
4. 合并内容

### 需要展开的内容
如果内容被折叠（如"展开全文"、"显示更多"）：
1. 用 Playwright 检测展开按钮
2. 点击展开
3. 重新获取内容

### 登录付费墙
如果遇到付费墙：
1. 提示用户内容需要付费/登录
2. 询问是否有账号信息
3. 如有，用 Playwright 完成登录流程
4. 如无，保存已获取的部分内容 + 原始 URL
