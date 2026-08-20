# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a personal Claude Code skills repository. Each skill is a self-contained directory that can be installed to `~/.claude/skills/` to extend Claude Code's capabilities.

## Repository Structure

```
ljg-skills/
├── ljg-*/              # Each skill is a directory with "ljg-" prefix
│   ├── SKILL.md        # Skill definition with YAML frontmatter
│   ├── references/     # Reference docs for complex skills
│   ├── assets/         # Templates, images, scripts
│   └── scripts/        # Helper scripts (bash, node)
├── README.md
└── .gitignore          # Ignores everything except ljg-*/ and specific files
```

## Skill Format

Each `SKILL.md` follows this structure:

```yaml
---
name: skill-name
description: "What this skill does. Use when user says..."
user_invocable: true|false
version: "x.x.x"
---

# Skill content in markdown...
```

## Skill Inventory

| Skill | Purpose | External Dependencies |
|-------|---------|----------------------|
| `ljg-card` | Content → PNG visuals (long cards, infographs, posters) | Node.js + Playwright |
| `ljg-paper` | Academic paper analysis pipeline | None |
| `ljg-plain` | Plain language rewriter | None |
| `ljg-word` | English word deep-dive | None |
| `ljg-writes` | Writing engine for thinking through ideas | None |
| `ljg-read` | Reading companion (伴读); all ask-nodes use AskUserQuestion option cards (见「本地交互偏好」) | None |

## Commands

### Install ljg-card Dependencies

`ljg-card` requires Playwright for screenshot capture:

```bash
cd ljg-card && npm install && npx playwright install chromium
```

### Install Skills (for users)

```bash
# Copy all skills to Claude Code
mkdir -p ~/.claude/skills
cp -r ljg-* ~/.claude/skills/
```

## 本地交互偏好（upstream 更新参考）

**这是用户对 skill 交互方式的固定偏好。upstream 更新 ljg-* / pyrojewel-* 相关 skill 时，若上游改动与此偏好冲突，保留本地方案；合并时按此参考决定取舍。**

**核心偏好：询问用「选项卡片」，不是干等打字。**

- 任何 skill 停下来问用户、或邀请用户表态/选择的地方，**默认用 AskUserQuestion 抛选项卡片**，而不是抛一句开放题干等用户敲字。
- 一次抛 1–3 个问题，每个给 **2–4 个口语、20 字内、可戳中的选项 + `Other`**（用户可手动输入）。
- 选项是 **探针，不是答案**——替用户把「此刻最可能的几种反应」说出来；用户点选后，skill 顺着那个选择深挖，不出现「点了 A 就算答对」。
- 每个选项带一行 `description`，说明这条路会引向哪里。
- `Other` **永远开放且优先**：用户主动手写永远优先于猜选项；选项只是方案，不是限制。
- `multiSelect` 仅当问题天然多选时开，默认单选。
- 规律性切换（快进/展开/继续）也选项化，但仍保留用户随时口头打断的通道。

**已应用此偏好的 skill：**
- `pyrojewel-deep-paper`（pyrojewel 生态）：全程 AskUserQuestion 批量深挖（见 pyrojewel_claude_code / skill-source-map）。
- `ljg-read`（本仓库）：4 个交互节点（注疏先问 / 碰撞提问 / 节奏切换 / 读后一句话）已全部选项化；「读后一句话」给提示方向选项，但那句本身必须由用户 `Other` 手写（不替读者省掉思考）。
- 后续新 skill 或上游 skill 改造，一律沿用此偏好。

**例外/边界**：真正必须读者自己生成、选项会替读者省掉思考的地方（如「读后一句话」、开放性复盘），选项只做脚手架（帮起头），核心内容仍由用户手写。

## Architecture Notes

### Skill Invocation

- Skills with `user_invocable: true` can be triggered via `/skill-name` or natural language
- Trigger phrases are defined in each skill's `description` field
- Skills can call other skills via the Skill tool

### Content Processing Pipeline

Several skills share a common pattern for content ingestion:
- **URL** → WebFetch
- **File path** → Read tool
- **Raw text** → Direct use

### ljg-card Architecture

The most complex skill with multiple rendering modes:

1. **HTML Templates**: Stored in `assets/` (long_template.html, infograph_template.html, poster_template.html)
2. **Capture Script**: `assets/capture.js` uses Playwright to screenshot HTML → PNG
3. **Reference Docs**: `references/taste.md` (design guidelines), `references/mode-*.md` (mode-specific instructions)
4. **Output**: PNG files written to `~/Downloads/`

### Shared Conventions

**Org-mode output** (ljg-paper, ljg-plain, ljg-writes):
- Bold: `*text*` (single asterisk, not `**`)
- Filenames: `{timestamp}--{title}__{type}.org`
- Output directory: `~/Documents/notes/`
- Timestamps: `date +%Y%m%dT%H%M%S`

**ASCII Art**:
- Allowed: `+ - | / \ > < v ^ * = ~ . : # [ ] ( ) _ , ; ! ' "`
- Forbidden: Unicode box-drawing characters

## Development Guidelines

- Skills are atomic units—each skill directory is self-contained
- Version numbers are manually maintained in SKILL.md frontmatter
- The `.gitignore` ignores all files by default; explicitly unignore with `!pattern`
- When modifying skill logic, update both the SKILL.md and any referenced files in `references/`

## Testing Changes

After modifying a skill:
1. Copy to `~/.claude/skills/`
2. Restart Claude Code to reload skills
3. Test via natural language trigger or `/skill-name`
