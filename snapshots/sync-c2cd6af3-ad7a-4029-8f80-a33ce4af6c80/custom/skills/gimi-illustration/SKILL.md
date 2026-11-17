---
name: 'gimi-illustration'
description: 'Generate quirky hand-drawn style illustrations from Chinese article text, with selectable IP characters including Gimi and 强少Qone. Use when the user invokes 配图, 给文章配图, 帮我配图, illustration, 强少Qone, qiangshao-qone, or similar.'
---

# gimi-illustration · 配图 Skill

> **Release v1.4** · 怪诞手绘单风格，IP：`gimi` / `qiangshao-qone` / `none`；公众号封面默认 2.35:1，正文配图默认 16:9。

输入文章内容，生成怪诞手绘风格封面或正文配图。单风格 `quirky-sketch`，IP 为 `gimi` / `qiangshao-qone` / `none`。

---

## 触发词

`配图` / `给文章配图` / `帮我配图` / `illustration`

---

## 参考文件（按需加载）

| 文件 | 何时加载 |
|------|---------|
| `references/styles/quirky-sketch.md` | 生图时 |
| `references/composition-patterns.md` | 出策略时 |
| `references/shot-config.md` | 出策略时（Shot List 模板） |
| `references/prompt-template.md` | 组装 prompt 时 |
| `references/prompt-examples.md` | 用户索要常用提示词；执行公众号封面、周刊封面、按章节配图或定向重做等常用场景时 |
| `references/qa-checklist.md` | 质检时 |
| `references/ip/gimi/ip.md` | `$IP` = gimi |
| `references/ip/qiangshao-qone/ip.md` | `$IP` = qiangshao-qone / 强少Qone |
| `references/ip/none.md` | `$IP` = none |

---

## 生图能力

调用当前 AI 软件**内置的生图工具**（如 Codex `@Image Gen`、Cursor `GenerateImage` 等）。Skill 负责策略、prompt 组装、参考图附件、QA；生图由平台工具执行。接口语义：文本 prompt + 可选图片附件。

平台适配优先级：

1. 支持显式图片附件的平台：把参考图作为 image reference 附件传入。
2. Codex `image_gen`：有 IP 时先把对应 `ip/{$IP}/ip.md` 要求的 **设定图 + 1 张校准图** 展示进上下文；再 `image_gen`
3. 不支持图片输入且无法把图片放入上下文时，暂停说明原因；只有用户接受降级时，才使用 `ip.md` 文字锚点描述。

---

## 核心变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `$STYLE` | `quirky-sketch` | v1.0 固定，不切换 |
| `$RATIO` | 按用途推断 | 公众号封面 → `2.35:1`；正文配图 → `16:9`；用户可覆盖 |
| `$IP` | `gimi` | `gimi` / `qiangshao-qone` / `none`；用户说“强少Qone”时映射为 `qiangshao-qone` |
| `$TITLE` | 无 | 用户要求时顶部手写标题 |
| `$COUNT` | 自动推断 | 配图张数 |
| `$OUTPUT_SIZE` | 无 | **可选**；仅用户明确要求具体像素时记录 |

**推荐像素（非强制）：** 公众号封面 2.35:1 → 建议 900×383；正文 16:9 → 建议 1920×1080；3:4 → 建议 1080×1440。

---

## 常用场景规则

### 公众号文章封面

- 默认比例：`2.35:1`；推荐尺寸：`900×383`
- 自动提炼全文最重要的 **1 个判断**，整张封面只表达该判断
- 自动提取 **3–5 个可画关键词**，优先级：具名产品/工具 > 文件/界面 > 设备/动作 > 抽象概念
- 选择 1 个关键词作为主视觉载体，其余只做辅助物件
- 不把章节目录或文章摘要平铺成信息图

### 公众号正文配图

- 默认比例：`16:9`
- 按一级/二级标题识别章节；无标题时按论点转折拆分
- 每章提炼 **1 个核心判断**，并提取 **2–5 个具象关键词**
- 每个章节默认生成 1 张；用户指定数量时，从最值得画的位置中选择
- 多张图的主构图、视角、角色动作和主载体不得重复
- 默认交付每张图的建议插入位置和 `shot-config.md`

### 共通约束

- 有 IP 时角色必须参与核心动作；若移除角色后画面含义不变，重写构图
- 优先画原文中的具名物件，不用灯泡、齿轮、大脑等泛化符号替代
- 每张只表达一个判断，中文标注 2–5 个短词
- 禁止 PPT 流程图、课程信息图、四宫格卡片和密集教程页

完整的可复制提示词见 `references/prompt-examples.md`。

---

## 工作流

### Quick Mode

用户已给正文且变量可推断 → 不多问，直接生图。以下情况才确认：用户说「先别生图」、变量冲突。

### Step 1 · 意图确认

记录：`$RATIO` · `$IP` · `$TITLE` · `$COUNT` ·（可选）`$OUTPUT_SIZE`

Quick Mode 推断：公众号文章/周刊封面 → 2.35:1（推荐 900×383）；竖版/小红书 → 3:4；公众号正文及其他横图 → 16:9。

### Step 2 · 消化内容 + 出策略

> 策略细节以 `composition-patterns.md` 为准；落盘格式以 `shot-config.md` 为准。

1. 加载上述两文件
2. 按 `composition-patterns.md` 的 **Step 2 决策流程** 执行（二轴 → 创意生成法六步 → 构图库 → Shot Config）
3. Quick Mode：默认直接落盘并进入 Step 3；用户说「先看策略」时先展示摘要

### Step 3 · 组装 Prompt + 生图

1. 加载 `styles/{$STYLE}.md`、`prompt-template.md`、对应 `ip/*.md`
2. `$IP` ≠ none → 按对应 `ip/{$IP}/ip.md` 执行参考图协议：
   - `$IP=gimi`：Gimi 设定图 + 对应主题校准图
   - `$IP=qiangshao-qone`：强少Qone 全身身份设定图 + 产品流程画风校准图
   - 所需参考图全部进入上下文后再生图
   - 若参考图未进上下文，不得声称已执行有 IP 流程
3. 填充变量槽，调用生图工具（`IP_DESC` 在 `STYLE_DNA` 之前，见 `prompt-template.md`）

### Step 4 · QA 质检

加载 `qa-checklist.md`。`$OUTPUT_SIZE` 有值时 → 等比缩放 + 白底留白，禁止裁切。不通过 → 重试，最多 2 次。

### Step 5 · 交付

```
outputs/{YYYYMMDD}-{slug}/
  shot-config.md      ← 配图策略 + 插入位置（与图片同时交付）
  01-{slug}.png
  02-{slug}.png       ← 多张时继续编号
```

---

## 版本

`v1.4` · 2026-07-27 · 新增公众号封面与按章节正文配图常用提示词；封面默认 2.35:1（900×383）