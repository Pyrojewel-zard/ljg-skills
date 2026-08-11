---
name: ljg-present
description: "演讲铸造器（Outline-Faithful）。把 markdown outline 1:1 铸成单文件离线 HTML；支持 black/red/yellow 与静态 hacker（--hacker，--cyber 兼容别名），自动处理标题封面、多行密度布局、表格、ASCII、LaTeX、自适应与翻页笔。USE WHEN 用户要求讲这个、present、做成演讲、slides、标语流、宣言体、slogan、manifesto、按 outline 美化。NOT FOR 内容提炼、改写或企业 PPT。"
user_invocable: true
version: "4.0.0"
---

# ljg-present：演讲铸造器

把 outline 铸成舞台。内容由作者决定，skill 只决定它如何被看见。

## 核心契约

**Outline 是真理，Skill 是渲染器。**

- 标题、段落、列表项、引用不改字。
- 表格不改结构，example/代码块不改空格与换行。
- 所有源元素按原顺序出现；不抽提、不浓缩、不重排。
- 唯一允许改变的是物理分页与视觉构图。
- YAML frontmatter 的 `title:` 或第一个 H1 是文档标题，必须先生成独立 cover；第一个 outline 节点仍在下一页。若两者文字完全相同，可合并为 cover，不能重复。

## Workflow Routing

| Workflow | Trigger | File |
|---|---|---|
| **Generate** | 讲这个、present、做成演讲、slides、按 outline 美化、生成 HTML 演示 | `Workflows/Generate.md` |

生成时先读 `RenderingSpec.md`，再使用根目录 `SloganTemplate.html`。不要根据记忆重造模板。

## Quick Reference

### 输入与输出

- 输入：Markdown 或纯文本。
- 输出：`~/Downloads/{title}.html`，单文件、离线、无外链资源。
- 首屏：文档标题 cover。
- Header：不承载任何信息。
- Footer：首页显示页码与 subtitle/meta；其他页只显示页码。

### Theme

优先级：显式参数 > YAML frontmatter 的 `tags:` > 默认 `black`。

| 参数 | theme | 调性 |
|---|---|---|
| `-b` / `--theme=black` | black | 沉思、论证 |
| `-r` / `--theme=red` | red | 宣言、号召 |
| `-y` / `--theme=yellow` | yellow | 反讽、警觉 |
| `--hacker` | hacker | 逆向工程实验纸 |
| `--cyber` | hacker | 兼容别名；不再生成 CRT/HUD |

Hacker 不是「整篇黑底绿字」。它只有三种主色：

```css
--hacker-void:   #07110D;
--hacker-paper:  #EAF4EC;
--hacker-signal: #00C46A;
```

普通页用浅色实验纸保证阅读；cover 与一级章节用深色场；信号绿只承担总线、切角、重点和表格标签。不要矩阵雨、发光描边、伪 HUD 或闪烁光标。

### Outline 映射

| Source | Page |
|---|---|
| `# 一级标题` | 独占 emphasis 章节页 |
| `##` 及更深标题 | 独占 title 页；深度越高字号越低 |
| 段落 | theme 文本页；仅在必要时物理拆页 |
| 列表 | 同一逻辑块 1–4 项同页，保留序号与 indent |
| 表格 | table 页；超过 6 行分页并重复表头 |
| 引用 | quote 页，保留原始段落关系 |
| fenced code | pre 页，逐字符保留 |
| `**强调**` / `` `code` `` | `hl: true`；emphasis 页忽略 inline hl |

### 多行不是统一降字号

多行页同时看「行数」和「文本密度」：

- 2 行：短/中等文本 → `duo`；高密度 → `rows`。
- 3 行：轻文本 → `triptych`；中/高密度 → `rows`。
- 4 行：`matrix`（2×2）；竖屏降级为单列。
- 网格项必须 `min-width: 0`，正文使用自然断词；不要把 `.line` 设成 flex/grid，以免拆散高亮与公式。

阈值和 DOM 字段以 `RenderingSpec.md` 为准。

### 公式、ASCII 与尺寸

- 只把闭合的 `$...$` / `$$...$$` 当作公式；`$20/month` 这类价格不是公式。
- 离线渲染常用符号、上下标，不依赖 MathJax/CDN。
- ASCII/pre 字号从 `3.6vmin` 起，再交给统一 fit guard 缩放。
- 每页都测量真实可用宽高；监听 resize、fullscreen、字体就绪和 ResizeObserver。
- `data-fits=true` 只证明没有越界；文本页若 `fitScale < 0.70`，必须重新拆页或换布局，不能以「缩进去了」冒充可读。

## 通用交互

- `→` `↓` `Space` `Enter` `j` `PageDown`：下一页。
- `←` `↑` `k` `PageUp`：上一页。
- `Home` / `End`：首末页。
- `f` / `F`：全屏。
- 触屏左右滑、点击左右半屏：翻页。

上下键与 PageUp/PageDown 同时保留，因为不同蓝牙翻页笔发送的键值不同。

## 验收门槛

写出 HTML 后运行：

```bash
bun Tools/ValidateDeck.ts ~/Downloads/<deck>.html --theme <theme>
```

Validator 负责静态契约：模板版本、JS 语法、标题 cover、header/footer、零动效、公式保护、多行布局、fit guard、页面类型、翻页键和外链资源。

视觉判断必须用 Interceptor 在隔离浏览器中复验典型页与高密度页。若隔离 context 不可用，报告「静态验证通过，尚未浏览器视觉复验」；不能改用主浏览器或其他截图工具，也不能宣称视觉已验证。

## Gotchas

- **Theme 不是配色别名。** 全局黑底会让长演示疲劳；Hacker 的信息层级来自浅/深场切换与结构线，不来自荧光特效数量。
- **最长行不能代表多行页。** 2、3、4 行在构图上是不同对象；只按最长字符降字号会把页面变成文字墙。
- **`vmin` 不是响应式。** 固定字号只能估算；真实边界必须由 `scrollWidth/scrollHeight` 与可用宽高共同计算。
- **`fits` 不等于可读。** 极端缩小仍可能得到 `fits=true`；低于 0.70 的文本 fitScale 是重新分页信号。
- **公式识别必须要求闭合 delimiter。** 否则价格、货币或路径中的 `$` 会被误判。
- **多行网格要设 `min-width: 0`。** 缺少它时，长词或公式会把列撑出 viewport。
- **`.line` 保持行内容器。** 将其设成 flex/grid 会拆开 chunks、inline math 与高亮；布局应作用于 `.lines`。
- **Header 与 footer 是不同契约。** Header 不放信息；meta 只在 cover footer，pager 每页都有。
- **禁止所有视觉动效。** 不只检查 shorthand，还要覆盖 `animation-*`、`transition-*`、`view-transition-*`、smooth scroll、`.animate()` 与定时器。
- **离线不能只扫 `<img>` 和 `https://`。** CSS 相对 `url(...)`、`@import`、`image-set(...)` 同样会让单文件在别的机器上缺资源。
- **真实浏览器证据不可替代。** 静态 validator 能阻止结构回归，但不能证明字体、换行和视觉节奏在真实 Chrome 中成立。

## Examples

### Example 1：常规 outline 演示

```text
User: 用 ljg-present 讲这个 notes/talk.md
→ 读取 Generate workflow、RenderingSpec 与 SloganTemplate
→ 保留全部 outline，生成标题 cover 与 black/red/yellow 主题页面
→ 运行 ValidateDeck，再输出 ~/Downloads/<title>.html
```

### Example 2：静态 Hacker 演示

```text
User: 把这篇 markdown 做成 Hacker style，不要动效
→ 选择 --hacker，普通页浅底、章节页深底
→ 按行数与密度生成 duo/triptych/rows/matrix
→ 验证零动效、公式、footer、自适应与翻页笔
```

## 中文默认

默认输出中文；原文是英文且用户要求保留时，不翻译。
