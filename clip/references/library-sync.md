# 资料库同步 — 统一管理界面

> 本文是 `/clip` 第五步 5.3「同步到资料库」的执行依据。执行本流程前需先调用 **@资料库** 技能，按其模块文档（`database/entry.md`、`page/import-flow.md`、`manage/entry.md`）的命令形态执行。

## 定位与架构

```
数据层（唯一事实源）：clip/collect.csv —— 本地每次剪藏追加一行（5.2 节）
        │
        ├─ 首次：导入资料库建表（database）→ 生成托管页（page，挂为数据表子节点）
        │
        └─ 每次：新增行增量写入资料库表 → 托管页订阅变更自动刷新

展示层（统一管理界面）：资料库托管的 collect.html
        通过只读 __SMART_PAGE__.database SDK 实时读表渲染
        搜索 / 筛选 / 排序 / 原文直达；HTML 本身不存数据、永不重写
```

职责边界：

- `clip/collect.csv` — 本地唯一事实源，Excel 可直接打开，同步失败时的兜底数据
- 资料库 database — 在线数据层，每次剪藏增量写入
- 资料库 page — 纯展示层，从模板生成一次后不再改动
- `clip/library-sync.json` — 同步状态，决定走"首次引导"还是"增量同步"

## 状态文件

路径：`clip/library-sync.json`

```json
{
  "spaceId": "<资料库空间 ID>",
  "csvNodeId": "<database 节点 id>",
  "databaseId": "<SDK 用的数据库 id，同 csvNodeId>",
  "htmlNodeId": "<page 节点 id>",
  "htmlUrl": "<托管页协作态链接>",
  "syncedAt": "YYYY-MM-DD"
}
```

- **不存在** → 走「首次引导」。
- **存在** → 走「每次增量同步」。
- 存在但字段缺失或已损坏 → 视同首次，但先向用户说明可能出现重复数据表，确认后再执行。

## 首次引导

前置：本轮 5.2 已确保 `clip/collect.csv` 存在。全程调用 @资料库 技能执行。

1. **导入 CSV 建表**：按 csv-import-flow 路径 A 执行
   `database/import_csv.py clip/collect.csv`
   - 目标空间：用户未指定时省略 `--space-id`，默认落"我的文档"（个人空间写入由用户的 clip 使用行为授权，无需再次确认；显式指定团队空间时按资料库变更规则先确认）
   - 成功取返回的 `node_block_id`（即 databaseId，SDK 使用的 id 就是源节点 id）
2. **查空间归属**：`space.workspace.node-info --node-id <csvNodeId>` 取 `spaceId`
3. **校验真实 schema**：用 database 模块的 schema 查询能力读取服务端真实字段名与类型，与预期字段核对：`id`(number) / `date`(date) / `title` / `author` / `source` / `category` / `word_count`(number) / `url` / `file`（后四项预期 text 或 url 类型，以后端推断为准）
   - 字段名与 CSV 表头一致 → FIELD_MAP 用恒等映射
   - 不一致 → 按真实字段名生成 FIELD_MAP，并记下每个字段的**真实类型**（增量写入时要按类型构造 PropertyValue）
4. **生成 collect.html**：从 `templates/collect.html.template` 渲染，替换两个占位符：
   - `__CLIP_DATABASE_ID__` → databaseId（字符串字面量）
   - `__CLIP_FIELD_MAP__` → FIELD_MAP 对象字面量（如 `{"id":"id","date":"date",...}`）
   - 替换后的临时 HTML 是中间产物，导入成功后删除
5. **上传托管**：按 page 模块 import-flow 执行
   `page/import_html.py <collect.html> --parent-id <csvNodeId> --space-id <spaceId> --databases '[{"id":"<databaseId>"}]'`
   - `--parent-id` 使 HTML 挂为数据表子节点；`--databases` 同时登记 page↔database 关联（页面 SDK 调用识别依赖它）
   - 模板无任何 `<img>` 标签，图片托管自检天然通过
6. **写状态文件**：从 `KS_IMPORT_OK` 提取 `node_block_id`、`url`，连同 spaceId、csvNodeId/databaseId、日期写入 `clip/library-sync.json`
7. **回执**：向用户给出托管页链接，说明"以后每篇剪藏会自动同步到这张表，打开此链接即为最新管理界面"

## 每次剪藏后的增量同步

前置：`clip/library-sync.json` 存在；本轮 5.2 刚向 `clip/collect.csv` 追加了行。

1. 取**本轮追加的行**（通常是 1 行；批量剪藏时为多行），按首次引导第 3 步记录的真实字段类型构造记录：
   - number 字段 → `{ number: <int> }`
   - date 字段 → `{ date: "YYYY-MM-DD" }`
   - text 字段 → `{ text: "<值>" }`
   - url 类型字段 → `{ url: { text: "原文", link: "<URL>" } }`；若推断为 text 则直接 `{ text: "<URL>" }`
2. 调用 `database/batch_add_database_records.py`（用法见 database/entry.md §3）写入，`databaseId` 取状态文件中的值
3. **失败降级（硬约束）**：
   - 任何资料库步骤失败都**不阻塞**本地剪藏主流程——笔记与 CSV 已落盘为准
   - 回执中说明"资料库同步失败（原因），可重试"
   - 重试只补写**未成功的行**，禁止整表重放（防重复记录）；无法确定哪些行已写入时，先查询资料库表再补差
4. 成功后无需回执链接（页面通过 `onUpdated` 订阅自动刷新）

## 降级与边界

- 首次引导在步骤 1–4 之间失败：未产生 page 节点，可直接重试；已建表则复用 `--database-id` 覆盖导入，避免重复建表
- 托管页在资料库外打开（如下载到本地）：SDK 不存在，页面显示引导文案"请在 WorkBuddy 资料库中打开"，属预期行为
- 文章笔记 md 始终存本地；托管页"链接"列只提供原文直达，笔记列已移除（本地相对路径在托管环境不可达）。如需在线阅读笔记，属后续扩展（笔记入库），不在本流程
- 不主动删除资料库任何节点；出现重复数据表时提示用户手动清理
