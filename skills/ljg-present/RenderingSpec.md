# Rendering Spec

`ljg-present` 的生成契约。`Workflows/Generate.md` 在处理任何演示前读取本文件；模板与 validator 以此为准。

## 1. Source Manifest First

解析时先建立 source manifest，再生成 slides。每个源元素获得稳定的 `SRC-NNN`，每张 slide 携带 `sourceIds`。

Manifest 顺序就是原稿顺序：

1. heading
2. paragraph
3. list item
4. quote
5. table caption / table
6. example / fenced code

生成后按 slides 的首次 `sourceIds` 引用去重，必须与 manifest 顺序一致。没有源元素可以静默消失。

## 2. Metadata, Cover and Chrome

| Source | Output |
|---|---|
| YAML `title:` / Markdown H1 | `<title>` 与 cover 主内容 |
| YAML `author:`、`date:` 或用户给出的 meta | subtitle/meta footer |
| YAML `tags:` | theme inference |

Cover 规则：

- YAML `title:` / H1 始终产生第 1 页 `cover:true`。
- 第一个 outline 节点保持为后续页面。
- 仅当第一个节点的纯文本与 title 完全相同，才把该节点升级为 cover，避免重复。
- Cover 不占 source manifest ID；合并时保留节点原 sourceId。

Chrome 规则：

- 不创建承载信息的 `<header>` 或顶部 guide。
- Footer pager 每页存在。
- Meta footer 仅 `index === 0` 可见；其余页面只显示 pager。

## 3. Page Types

```jsonc
{
  "cover": true,
  "emphasis": true,
  "title": true,
  "depth": 2,
  "quote": true,
  "sourceIds": ["SRC-001"],
  "lines": [
    {"indent": 0, "chunks": [{"t": "文本"}, {"t": "重点", "hl": true}]}
  ]
}
```

```jsonc
{
  "preTitle": "optional",
  "pre": "ASCII / code，逐字符保留",
  "sourceIds": ["SRC-002"]
}
```

```jsonc
{
  "table": {
    "caption": "optional",
    "header": true,
    "rows": [["A", "B"], ["C", "D"]]
  },
  "sourceIds": ["SRC-003", "SRC-004"]
}
```

`table.header` 由 Markdown separator row 推断。不要无条件把第一行当表头；那会改变无表头表格的含义。

## 4. Theme Grammar

一篇演示只使用一个 theme。

| theme | regular | cover / emphasis | hl |
|---|---|---|---|
| black | 黑底白字 | 红底白字 | 红 |
| red | 红底白字 | 黑底白字 | 金 |
| yellow | 黄底黑字 | 黑底白字 | 红 |
| hacker | `#EAF4EC` 纸面 / `#07110D` 字 | `#07110D` 场 / `#EAF4EC` 字 | `#00C46A` |

Hacker 的生成语法：

- 左侧不对称信号总线建立结构。
- 静态细网格只作为纸面刻度，不做全屏 HUD。
- Cut corner、硬边、表格标签可使用 signal green。
- ASCII/pre 使用深色硬边面板。
- 无阴影漂浮、无 Matrix 雨、无闪烁、无 glow 堆叠。

## 5. Text Length and Multi-line Density

CJK 字符按 `1.8`，其他字符按 `1` 计权。

```js
const weights = slide.lines.map(lineLength);
const lineCount = slide.lines.length;
const maxWeight = Math.max(...weights);
const totalWeight = weights.reduce((sum, value) => sum + value, 0);

const density =
  maxWeight >= 45 || totalWeight >= 110 ? "dense" :
  maxWeight >= 24 || totalWeight >= 65 ? "medium" :
  "light";

const layout =
  lineCount === 2
    ? (maxWeight >= 38 || totalWeight >= 90 ? "rows" : "duo")
    : lineCount === 3
      ? (maxWeight <= 18 && totalWeight <= 55 ? "triptych" : "rows")
      : lineCount === 4
        ? "matrix"
        : "single";
```

写入：

- slide：`data-line-count`、`data-density`、`data-layout`
- line：`data-line-index`、`data-weight`

CSS 顺序：单行 `data-len` 规则在前，多行规则在后。多行 `.lines` 才变成 grid；`.line` 保持普通 inline content 容器。

响应式：portrait 下 `duo`、`triptych`、`matrix` 全部变成单列，随后重新执行 fit。

## 6. Pretty Wrapping

- 单行短文本优先 `nowrap`，交给 fit guard。
- `xlong`、quote、多行卡片允许换行。
- 使用 `overflow-wrap: break-word`、`word-break: normal`、`text-wrap: pretty`。
- 禁止 `overflow-wrap: anywhere` 作为默认，它会产生难看的中文/英文断裂。
- Grid item 必须 `min-width: 0; min-height: 0`。

## 7. Measured Fit

所有可见内容放在一个 `.fit-box` 中。每次显示页面时：

1. 清除旧 transform。
2. 从 slide computed style 取四边 padding。
3. 计算 `availableWidth/availableHeight`。
4. 读取 `.fit-box.scrollWidth/scrollHeight`。
5. `scale = min(1, availableWidth/contentWidth, availableHeight/contentHeight) * 0.975`。
6. 写入 `data-fit-scale` 与 `data-fits`。

重新测量触发：

- `resize`
- `fullscreenchange`
- `document.fonts.ready`
- `ResizeObserver`（能力检测后启用）
- portrait media query 改变布局后

文本页 `fitScale < 0.70` 视为可读性失败：先拆页或换 rows，再考虑缩字。ASCII 与大表格可低于 0.70，但必须人工确认。

## 8. Offline Math

只解析闭合 delimiter；inline 起始 `$` 后不能直接是数字或问号：

- display：完整的 `$$...$$` 或整行 `$...$`
- inline：句子中的闭合 `$...$`

最低支持：

- `\cdot`、`\times`、`\propto`
- `\alpha`、`\beta`、`\gamma`
- `^{...}` / `^x`
- `_{...}` / `_x`

不认识的命令原样保留。`$20/month`、`$200/month`、`$???/month` 必须保持普通文本，即使多个价格出现在同一行也不能彼此配成公式。数字开头的数学表达式使用 `$$...$$`。

## 9. Motion and Interaction

全篇硬切。禁止：

- CSS `animation*` / `transition*` / `view-transition*` / `@keyframes`
- `scroll-behavior: smooth`
- JS `.animate()` / `setInterval()`
- 闪烁 cursor 或自动播放

允许 `requestAnimationFrame` 仅用于布局测量与 paint 后 fit，不用于视觉运动。

键盘映射：

| Action | Keys |
|---|---|
| next | ArrowRight, ArrowDown, Space, Enter, j, PageDown |
| prev | ArrowLeft, ArrowUp, k, PageUp |
| first / last | Home / End |
| fullscreen | f / F |

## 10. Verification

静态验证同时拒绝资源标签、`@import`、CSS `url(...)` 与 `image-set(...)`，确保单文件真正离线：

```bash
bun Tools/ValidateDeck.ts <html> --theme <theme>
```

视觉验证：使用 Interceptor 的隔离测试 context，至少检查：

- cover
- 普通单行页
- 最长单段页
- 2/3/4 行中密度最高的页面
- 最大表格
- 最大 ASCII/pre
- 每一种公式
- landscape 与 portrait 各一种尺寸

Interceptor 不可用时，保留静态证据并明确标记 deferred；不要用其他浏览器自动化或主浏览器替代。
