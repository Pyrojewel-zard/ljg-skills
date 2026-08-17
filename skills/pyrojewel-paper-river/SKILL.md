---
name: pyrojewel-paper-river
description: "论文溯源倒读法：递归找出前序论文（最多5层），正向讲述问题演化史。Trigger: '论文溯源', '论文脉络', 'pyrojewel-paper-river', 'paper river'. Supports Zotero MCP with automatic image asset management."
trigger:
  - "论文溯源"
  - "论文脉络"
  - "pyrojewel-paper-river"
  - "paper river"
  - "倒读法"
user_invocable: true
version: "1.3.0"
---

# pyrojewel-paper-river: 倒读法溯源

## Constants

- **ZOTERO_MARKDOWN_PATH** = `{zotero_markdown_path}` (from .claude/settings.json env)
- **DEFAULT_OUTPUT_DIR** = `{project}/notes/papers/`
- **PROJECT_PAPER_OVERVIEW_DIR** = `{project}/paper_overview/` (if this directory exists)
- **DIRECTION** = required or inferred direction slug, e.g. `vf-literature`, `transformer-modeling`, `inverse-synthesis`
- **OUTPUT_DIR** =
  - paper_overview mode: `{project}/paper_overview/rivers/{DIRECTION}/`
  - legacy mode: `{project}/notes/papers/`
- **SOURCES_DIR** =
  - paper_overview mode: `{project}/paper_overview/sources/`
  - legacy mode: `{OUTPUT_DIR}/sources/`
- **IMAGES_DIR** =
  - paper_overview mode: `{project}/paper_overview/sources/{paper_id_or_attachmentKey}/images/`
  - legacy mode: `{OUTPUT_DIR}/images/{attachmentKey}/`

### Project-aware output rule

If `{project}/paper_overview/` exists, this skill **must** use paper_overview mode. Do not write final river outputs to `{project}/raw/paper/`, `{project}/notes/papers/`, `autoModel/raw/paper/`, `autoModel/paper_Read/`, or `autoModel/idea-stage/`.

paper_overview mode separates assets by role:

```text
paper_overview/
├── sources/<paper_id_or_attachmentKey>/content.md
├── sources/<paper_id_or_attachmentKey>/images/*.jpeg
├── readings/<DIRECTION>/*.md
├── rivers/<DIRECTION>/<timestamp>--paper-river-<topic>.md
└── indexes/river_registry.json
```

`DIRECTION` must be a stable slug. If the user did not provide one and it cannot be inferred from the task, ask for it before writing files.

---

一篇论文不是孤岛。它站在前人的肩上，也踩着前人的伤疤。倒着挖到根，再正着看过来——问题怎么长出来的，每个人看到了什么别人没看到的，解法怎么一步步逼近真相。

## 核心逻辑

读论文最常见的错：只看眼前这一篇，不知道它从哪来。倒读法反过来——先找到这篇论文在批判谁、改进谁，再找那篇论文又在批判谁，递归五层，挖到源头。然后掉头，从源头正向读回来。

这样读完，你拿到的不是一篇论文的知识，是一整条问题演化线的理解。

## 格式约束

### Markdown 语法

- 加粗用 `**bold**`（双星号）
- 标题层级从 `#` 开始，不跳级
- 代码块用 ` ``` ` 包裹

### ASCII Art

所有图表用纯 ASCII 字符。允许：`+ - | / \ > < v ^ * = ~ . : # [ ] ( ) _ , ; ! ' "` 和空格。禁止 Unicode 绘图符号。

### 输出目录

- 输出目录：**$OUTPUT_DIR**（若不存在则 `mkdir -p`）
- paper_overview mode 图片目录：`paper_overview/sources/{paper_id_or_attachmentKey}/images/`
- paper_overview mode 原文备份：`paper_overview/sources/{paper_id_or_attachmentKey}/content.md`
- legacy mode 图片目录：`$OUTPUT_DIR/images/{attachmentKey}/`
- legacy mode 原文备份：`$OUTPUT_DIR/sources/{attachmentKey}.md`

### 文件命名规范

- 时间戳：`date +%Y%m%dT%H%M%S`
- 可读时间：`date "+%Y-%m-%d %a %H:%M"`
- 文件名：`{时间戳}--paper-river-{简短标题}.md`

### Markdown 文件头（YAML frontmatter）

```yaml
---
title: paper-river-{简短标题}
date: {YYYY-MM-DD HH:MM}
tags: [paper, river]
direction: {DIRECTION}
identifier: {YYYYMMDDTHHMMSS}
source: {URL 或来源描述}
authors: {目标论文作者}
venue: {发表场所/年份}
seed_paper: {paper_id_or_attachmentKey}
canonical_sources:
  - paper_overview/sources/{paper_id_or_attachmentKey}/
---
```

## 红线

1. *问题为轴* — 整篇文章的主线是"问题怎么演化的"，不是"论文怎么排列的"。论文是配角，问题是主角
2. *口语检验* — 你会这样跟朋友讲一个领域的发展史吗？不会就改
3. *差异为核* — 每篇论文的讲解重心是"它和前一篇的差异在哪"，不是独立地介绍每篇论文
4. *零术语* — 先用大白话落地，再顺带提术语名
5. *逻辑不断链* — 从第一篇到最后一篇，因果链条不能断。读者能感受到"所以他们才会这样做"
6. *诚实* — 找不到五层就说找到几层。论文之间的关系不确定就说不确定。不编造引用关系

## 写作原则

1. *差异驱动叙事* — 不要给每篇论文写独立摘要再拼起来。以"这篇看到了前一篇的什么问题"作为每段的开头，让差异本身推动叙事往前走
2. *变形替代定义* — 讲两个方案的区别时，把方案A连续变形成方案B。"如果你把X去掉，再加上Y，你就得到了Z"——比"Z和X的区别是..."有力十倍
3. *推理外显* — 每个解法出现前，先让读者感受到"不这么做不行了"的压力。模拟发现的过程，不是汇报发现的结果
4. *一张图胜千言* — 在演化叙事之前画溯源地图，在叙事之后画压缩总览图。让读者先有全景再入细节，细节看完再回全景

## 执行

### 1. 获取目标论文

按以下优先级**串行**获取论文全文（必须等待前一步完成再执行下一步）：

| 优先级 | 输入类型 | 获取方式 |
|--------|---------|---------|
| **1a** | Zotero（有 content.md） | `search_library` → `get_item_details` 获取 attachment key → Read content.md + Read 关键图片 |
| **1b** | Zotero（无 content.md） | `search_library` → `get_content` 纯文本 |
| **2** | IEEE 论文 | `/ieee-get-fulltext` **串行执行** |
| **3** | arxiv URL | WebFetch 访问 `arxiv.org/html/...` |
| **4** | PDF | Read |

**禁止使用 WebSearch 获取论文全文**。WebSearch 仅用于查找引用关系和后续论文线索，不用于直接获取论文内容。

**Zotero Markdown 优先路径**：search_library → get_item_details → 拼接 `$ZOTERO_MARKDOWN_PATH/<attachmentKey>/content.md` → ls 验证 → 存在则 Read（保留图片引用），不存在则回退 get_content。

**图片理解（content.md 存在时）**：扫描 `![](xxx.jpeg)` 引用，一律用 vision-batch-read skill 批量并发读图（淘汰 Read 逐张读）。记图注和内容描述。每张引用到笔记的图都必须经过实际读图。

**递归溯源中的图片资产管理（pyrojewel新增）**：

每递归到一篇新论文时，如果该论文来自Zotero（有attachmentKey）：
1. paper_overview mode: `mkdir -p paper_overview/sources/{paper_id_or_attachmentKey}/images`
2. paper_overview mode: `cp $ZOTERO_MARKDOWN_PATH/{attachmentKey}/*.{jpeg,png,jpg} paper_overview/sources/{paper_id_or_attachmentKey}/images/ 2>/dev/null; true`
3. paper_overview mode: `cp $ZOTERO_MARKDOWN_PATH/{attachmentKey}/content.md paper_overview/sources/{paper_id_or_attachmentKey}/content.md`
4. paper_overview mode 笔记中引用该论文图片：`../../sources/{paper_id_or_attachmentKey}/images/xxx.jpeg`（river 文件位于 `paper_overview/rivers/<DIRECTION>/`）
5. legacy mode 仍可使用 `images/{attachmentKey}/xxx.jpeg`
6. 不同论文的图片按 paper_id 或 attachmentKey 分区，不会冲突

**IEEE 论文检索必须串行**：
- `/ieee-get-fulltext` 涉及浏览器自动化，必须等待导航完成（约 20 秒）
- 禁止并行调用多个 IEEE 检索
- 递归溯源时，每篇论文串行处理

确保拿到：标题、作者、摘要、引言（尤其是 related work / introduction 中对前人工作的批判）。

### 2. 提取批判链线索

仔细读目标论文的引言和相关工作部分。找出：

- 它明确说"前人方法 X 有问题 Y"的地方
- 它声称自己改进了哪篇/哪几篇论文
- 它对比的 baseline 是谁

从中锁定 *被批判/被改进的核心论文*（通常 1-3 篇，选最直接的那条线）。

### 3. 递归溯源（深度研究）

对第 2 步找到的核心前序论文，重复同样的过程：它又在批判谁？改进谁？

递归规则：
- 最多递归 5 层（到第 5 层或到该领域的奠基论文为止）
- 每层只追 *问题最相关的那条线*，不发散
- 如果某层找不到明确的被批判对象，停在那里

**每递归一层后暂停**：展示当前溯源深度和已找到的论文链，询问用户"继续溯源下一层？还是当前深度够了？"用户说"够了"→停止递归，直接进入步骤5。

**提取批判链的具体操作**：
1. 扫描引言段落的"However"/"limitation"/"problem with"/"previous approach"等信号词
2. 找到批判语句后，提取被批判的方法名/论文引用编号
3. 从References列表中定位对应论文的完整信息（标题、作者、年份）
4. 对每篇前序论文，在引言/related work中同样操作，递归挖掘

**Zotero 语义搜索发现相关论文**：

使用 `mcp__zotero-mcp__semantic_search` 发现 Zotero 库中与当前论文语义相关的其他论文：

```
mcp__zotero-mcp__semantic_search({
  query: "{论文核心方法/问题关键词}",
  limit: 15
})
```

语义搜索的优势：
- 基于论文内容的向量相似度，而非关键词匹配
- 能发现标题不同但方法相似的论文
- 适合发现同一问题域的演化线索

搜索策略：
1. 用目标论文的核心方法名搜索（如 "invertible neural network inverse design"）
2. 用问题域关键词搜索（如 "microwave inverse modeling"）
3. 用理论基础搜索（如 "normalizing flow", "real NVP"）

对每篇发现的论文，优先检查 content.md 是否存在（走 Zotero Markdown 优先路径），不存在则用 `mcp__zotero-mcp__get_content` 获取纯文本，确认是否与演化线相关。

### 4. 前沿延伸

反方向：目标论文之后，有没有新论文在批判/改进它？

**Zotero 语义搜索后续工作**：

```
mcp__zotero-mcp__semantic_search({
  query: "{目标论文核心方法} improvement extension",
  limit: 15
})
```

同样用 `mcp__zotero-mcp__search_library` 搜索引用了目标论文的后续工作。

找到最相关的 1-3 篇后续论文，获取同样的信息。

### 5. 构建演化线

把第 3、4 步的结果整理成时间线：

```
[最老] Paper_0 → Paper_1 → ... → [目标论文] → [后续论文]
```

每条箭头标注：后者看到了前者的什么问题。

### 6. 正向费曼叙事

从最老的论文开始，正向讲述。关键：不是逐篇独立介绍，而是以问题演化为线索串联。

每篇论文讲三件事（以差异为重心）：
1. 它看到了前人方案的什么具体问题（用例子或场景说明）
2. 它的解法核心思路（用类比讲清楚）
3. 这个解法又留下了什么新的问题（自然过渡到下一篇）

### 7. 画图

两张图：
- *溯源地图*：放在演化叙事之前，展示论文间的引用/批判关系
- *问题-解法总览*：放在叙事之后，把整条线压缩到一屏。让人扫一眼就知道这条线怎么长出来的

### 8. 提炼洞见

读完整条线，回答：
- 这条演化线背后真正在发生什么变化？（不是表面的技术迭代，是更深层的认知转变）
- 下一步最可能往哪走？

### 9. 过红线 + 生成文件

逐条扫红线。额外检查：
- 因果链条是否连贯——把所有"它看到了什么问题"串起来读，逻辑通不通
- 差异是否突出——每篇论文的重点是不是在讲"和前面有什么不同"

### 10. 生成 Markdown 文件

1. 获取时间戳：`date +%Y%m%dT%H%M%S`
2. 确定输出目录：
   - paper_overview mode: `{当前项目目录}/paper_overview/rivers/{DIRECTION}/`
   - legacy mode: `{当前项目目录}/notes/papers/`
3. 按格式约束写入 Markdown 文件
4. 报告文件路径给用户

## 验收

- *问题是主角*：读完后记住的是"问题怎么演化的"，不是"有哪些论文"
- *因果不断*：从第一篇到最后一篇，每个转折都有"所以"
- *差异清晰*：每篇论文的独特贡献一句话能说清
- *外行能跟*：不懂这个领域的聪明人读完能复述这条演化线
- *两张图能独立看*：不读正文，只看图也能抓住大意
- *诚实标注*：哪些是确认的引用关系，哪些是推测的，标清楚
