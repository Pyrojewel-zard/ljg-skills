# Generate Workflow

把 Markdown 或纯文本铸成 outline-faithful 单文件 HTML 演示。

## 1. Announce

```bash
curl -s -X POST http://localhost:31337/notify \
  -H "Content-Type: application/json" \
  -d '{"message":"Running the Generate workflow in the ljg-present skill"}' \
  >/dev/null 2>&1 &
```

输出：`Running the **Generate** workflow in the **ljg-present** skill to build an outline-faithful HTML presentation...`

## 2. Load the Contract

完整读取：

1. `RenderingSpec.md`
2. `SloganTemplate.html`

不要从记忆重写模板，也不要复制上一次生成的 HTML 当新模板。

## 3. Read and Inventory the Source

- 读取完整输入，不只读前几百行。
- 提取 title、subtitle/meta、filetags。
- 建立稳定 `SRC-NNN` source manifest。
- 统计 heading、paragraph、list item、quote、table、example 数量。
- 记录每个源元素的原文与顺序；这是生成后的 fidelity oracle。

URL 输入先获取正文；本地文件优先直接读取。不能获取完整内容时停止并说明，不得凭摘要补写。

## 4. Resolve Theme

优先级：显式参数 > filetags > black。

| User intent | Theme |
|---|---|
| `-r`, `--theme=red` | red |
| `-b`, `--theme=black` | black |
| `-y`, `--theme=yellow` | yellow |
| `--hacker`, Hacker style | hacker |
| `--cyber` | hacker（兼容别名） |
| `:share:`, `:talk:`, `:manifesto:`, `:keynote:` | red |
| `:critique:`, `:warn:`, `:rant:` | yellow |
| 其他 | black |

若用户给出自定义视觉方向，先把它翻译成阅读策略（regular/cover/hl/结构线），再改主题变量；不要只堆效果词。

## 5. Parse into Slides

按 `RenderingSpec.md` 映射：

- Title cover 独立于 outline。
- 一级标题 → emphasis。
- 二级及更深标题 → title 页。
- 段落、列表、引用 → lines。
- table → `table`，显式写 `header: true|false`。
- example/fenced code → `pre`。
- 每张 slide 写 `sourceIds`。

分页时只切物理边界，不改文字。长段落优先按原有句界拆；同源拆分页保持同一字号、缩进和主题。

## 6. Build the HTML

从 `SloganTemplate.html` 替换四个占位符：

| Placeholder | Value |
|---|---|
| `{{TITLE}}` | HTML-escaped document title |
| `{{SUBTITLE}}` | HTML-escaped meta，缺失则空 |
| `{{THEME}}` | black/red/yellow/hacker |
| `{{SLIDES_JSON}}` | safe JSON serialization |

写到 `~/Downloads/{title}.html`。文件名去除路径字符，保留可读中文，控制在 40 字符内。

## 7. Fidelity Audit

在写出后立即检查：

1. slides 的首次 `sourceIds` 去重顺序与 source manifest 完全一致。
2. 每个 source ID 至少被引用一次。
3. 表格单元格、example 空白与公式原文保持不变。
4. Cover title 正确，原第一个节点仍存在。

任何漏项、重复消费或顺序漂移都先修解析器，不在输出末尾补页。

## 8. Static Validation

```bash
bun Tools/ValidateDeck.ts ~/Downloads/{title}.html --theme <theme>
```

失败即停止交付并修复。不要删掉 validator 不喜欢的规则来换 PASS；检查它指出的契约是否真的被破坏。

## 9. Visual Verification

用 Interceptor 隔离测试 profile 打开本地 HTML，执行 DOM、console、network、screenshot 四探针，并检查 `RenderingSpec.md` 列出的代表页。

- Interceptor gate 失败：停止浏览器流程，保留静态验证结果，报告「未浏览器复验」。
- 不触碰 Default profile。
- 不以 headless browser、系统截图或主浏览器替代。

## 10. Report

返回：

- HTML 的绝对路径。
- theme 与页数。
- source fidelity 结果。
- validator 结果。
- 浏览器视觉验证结果或 deferred 原因。
- 翻页键：`→ ← ↑ ↓ Space PageUp PageDown F Home End`。
