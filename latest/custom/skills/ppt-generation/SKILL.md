---
name: 'ppt-generation'
description: '当用户要求生成、创建或制作演示文稿（PPT/PPTX）时使用此技能。通过为每张幻灯片生成 AI 图像并将其组合成 PowerPoint 文件来创建视觉丰富的演示文稿。'
---

# PPT 生成技能

## 概述

此技能通过为每张幻灯片生成 AI 图像并将其组合成 PPTX 文件来生成专业的 PowerPoint 演示文稿。工作流程包括规划具有一致视觉风格的演示文稿结构、按顺序生成幻灯片图像（使用前一张幻灯片作为风格一致性的参考），并将它们组装成最终的演示文稿。

## 核心能力

- 规划和构建具有统一视觉风格的多页演示文稿
- 支持多种演示风格：商务、学术、简约、苹果主题演讲、创意
- 使用图像生成技能为每张幻灯片生成独特的 AI 图像
- 通过使用前一张幻灯片作为参考图像来保持视觉一致性
- 将图像组合成专业的 PPTX 文件

## 演示风格

创建演示计划时选择以下风格之一：

| 风格 | 描述 | 适用场景 |
|------|------|---------|
| **glassmorphism（毛玻璃）** | 模糊效果的磨砂玻璃面板、浮动半透明卡片、充满活力的渐变背景、通过层次感营造深度 | 科技产品、AI/SaaS 演示、未来感推介 |
| **dark-premium（暗色高级）** | 丰富的黑色背景 (#0a0a0a)、发光的强调色、微妙的光晕效果、奢华品牌美学 | 高端产品、高管演示、奢侈品牌 |
| **gradient-modern（渐变现代）** | 大胆的网格渐变、流畅的色彩过渡、当代排版、充满活力而精致 | 初创公司、创意机构、品牌发布 |
| **neo-brutalist（新粗野主义）** | 粗犷大胆的排版、高对比度、刻意的"丑陋"美学、反设计作为设计、孟菲斯风格 | 前卫品牌、Z 世代定位、颠覆性初创公司 |
| **3d-isometric（3D 等距）** | 干净的等距插图、浮动的 3D 元素、柔和阴影、科技前沿美学 | 科技解说、产品功能、SaaS 演示 |
| **editorial（编辑）** | 杂志品质的布局、精致的排版层次、戏剧性的摄影、Vogue/Bloomberg 美学 | 年度报告、奢侈品牌、思想领导力 |
| **minimal-swiss（极简瑞士）** | 基于网格的精度、Helvetica 风格排版、大胆运用留白、永恒的现代主义 | 建筑、设计公司、高端咨询 |
| **keynote（主题演讲）** | 苹果风格美学，大胆排版、戏剧性图像、高对比度、电影感 | 主题演讲、产品发布、励志演讲 |

## 工作流程

### 步骤 1：了解需求

当用户请求生成演示文稿时，确定：

- 主题/主题：演示文稿关于什么
- 幻灯片数量：需要多少张幻灯片（默认：5-10 张）
- **风格**：商务 / 学术 / 极简 / 主题演讲 / 创意
- 宽高比：标准（16:9）或经典（4:3）
- 内容大纲：每张幻灯片的要点

### 步骤 2：创建演示计划

在 `/mnt/user-data/workspace/` 中创建包含演示结构的 JSON 文件。**重要**：包含 `style` 字段以定义整体视觉一致性。

```json
{
  "title": "演示标题",
  "style": "keynote",
  "style_guidelines": {
    "color_palette": "深黑色背景、白色文本、单一强调色（蓝色或橙色）",
    "typography": "粗体无衬线标题、干净的正文、戏剧性的大小对比",
    "imagery": "高质量摄影、全出血图像、电影级构图",
    "layout": "充足的留白、居中焦点、每张幻灯片元素最少化"
  },
  "aspect_ratio": "16:9",
  "slides": [
    {
      "slide_number": 1,
      "type": "title",
      "title": "主标题",
      "subtitle": "副标题或标语",
      "visual_description": "用于图像生成的详细描述"
    },
    {
      "slide_number": 2,
      "type": "content",
      "title": "幻灯片标题",
      "key_points": ["要点 1", "要点 2", "要点 3"],
      "visual_description": "用于图像生成的详细描述"
    }
  ]
}
```

### 步骤 3：按顺序生成幻灯片图像

**重要**：**严格按顺序逐一生成幻灯片**。不要并行化或批量生成图像。每张幻灯片都依赖于前一张幻灯片的输出作为参考图像。并行生成幻灯片会破坏视觉一致性，这是不允许的。

1. 读取图像生成技能：`/mnt/skills/public/image-generation/SKILL.md`

2. **对于第一张幻灯片（幻灯片 1）**，创建建立视觉风格的提示：

```json
{
  "prompt": "专业演示幻灯片。[计划中的 style_guidelines]。标题：'您的标题'。[visual_description]。此幻灯片为整个演示建立视觉语言。",
  "style": "[基于选择的风格 - 例如苹果主题演讲美学、戏剧性灯光、电影感]",
  "composition": "清晰的文本层次结构的干净布局，[特定风格的构图]",
  "color_palette": "[来自 style_guidelines]",
  "typography": "[来自 style_guidelines]"
}
```

```bash
python /mnt/skills/public/image-generation/scripts/generate.py \
  --prompt-file /mnt/user-data/workspace/slide-01-prompt.json \
  --output-file /mnt/user-data/outputs/slide-01.jpg \
  --aspect-ratio 16:9
```

3. **对于后续幻灯片（幻灯片 2+）**，使用前一张幻灯片作为参考图像：

```json
{
  "prompt": "延续参考图像视觉风格的专业演示幻灯片。保持相同的调色板、排版风格和整体美学。标题：'幻灯片标题'。[visual_description]。保持与参考图像的视觉一致性。",
  "style": "完全匹配参考图像的风格",
  "composition": "与参考图像类似的布局原则，适配此内容",
  "color_palette": "与参考图像相同",
  "consistency_note": "此幻灯片必须看起来与参考图像属于同一演示文稿"
}
```

```bash
python /mnt/skills/public/image-generation/scripts/generate.py \
  --prompt-file /mnt/user-data/workspace/slide-02-prompt.json \
  --reference-images /mnt/user-data/outputs/slide-01.jpg \
  --output-file /mnt/user-data/outputs/slide-02.jpg \
  --aspect-ratio 16:9
```

4. **继续处理所有剩余幻灯片**，始终引用前一张幻灯片：

```bash
# 幻灯片 3 引用幻灯片 2
python /mnt/skills/public/image-generation/scripts/generate.py \
  --prompt-file /mnt/user-data/workspace/slide-03-prompt.json \
  --reference-images /mnt/user-data/outputs/slide-02.jpg \
  --output-file /mnt/user-data/outputs/slide-03.jpg \
  --aspect-ratio 16:9

# 幻灯片 4 引用幻灯片 3
python /mnt/skills/public/image-generation/scripts/generate.py \
  --prompt-file /mnt/user-data/workspace/slide-04-prompt.json \
  --reference-images /mnt/user-data/outputs/slide-03.jpg \
  --output-file /mnt/user-data/outputs/slide-04.jpg \
  --aspect-ratio 16:9
```

### 步骤 4：组合 PPT

所有幻灯片图像生成后，调用组合脚本：

```bash
python /mnt/skills/public/ppt-generation/scripts/generate.py \
  --plan-file /mnt/user-data/workspace/presentation-plan.json \
  --slide-images /mnt/user-data/outputs/slide-01.jpg /mnt/user-data/outputs/slide-02.jpg /mnt/user-data/outputs/slide-03.jpg \
  --output-file /mnt/user-data/outputs/presentation.pptx
```

参数：

- `--plan-file`：演示计划 JSON 文件的绝对路径（必需）
- `--slide-images`：按顺序排列的幻灯片图像绝对路径（必需，空格分隔）
- `--output-file`：输出 PPTX 文件的绝对路径（必需）

[!注意]
不要读取 Python 文件，只需使用参数调用它。

## 完整示例：毛玻璃风格（最现代前卫）

用户请求："创建一个关于 AI 产品发布的演示文稿"

### 步骤 1：创建演示计划

创建 `/mnt/user-data/workspace/ai-product-plan.json`：
```json
{
  "title": "Nova AI 发布",
  "style": "glassmorphism",
  "style_guidelines": {
    "color_palette": "充满活力的紫色到青色渐变背景（#667eea→#00d4ff），磨砂玻璃面板 15-20% 白色不透明度，电子强调色",
    "typography": "SF Pro Display 风格，粗体 700 权重白色标题带微妙文字阴影，干净的 400 权重正文，玻璃上出色的对比度",
    "imagery": "抽象 3D 玻璃球体、浮动半透明几何形状、柔和发光球体、通过分层透明度营造深度",
    "layout": "居中磨砂玻璃卡片带 32px 圆角、48-64px 内边距、浮动于渐变之上、柔和阴影的层次深度",
    "effects": "玻璃面板 20-40px 背景模糊、微妙白色边框光晕、匹配渐变的柔和彩色阴影、光线折射效果",
    "visual_language": "苹果 visionOS 美学、通过透明度的高级深度、未来感而亲和、2024 设计趋势"
  },
  "aspect_ratio": "16:9",
  "slides": [
    {
      "slide_number": 1,
      "type": "title",
      "title": "Nova AI 发布",
      "subtitle": "智能，重新定义",
      "visual_description": "从深紫色（#667eea）经品红到青色（#00d4ff）的惊艳渐变背景。中央：带强背景模糊的大型磨砂玻璃面板，包含粗体白色标题 'Nova AI 发布' 和较轻的副标题。浮动 3D 玻璃球体和抽象形状围绕卡片营造深度。玻璃面板后散发柔和光晕。高级 visionOS 美学。玻璃卡片带微妙白色边框（1px rgba 255,255,255,0.3）和柔和紫色调阴影。"
    },
    {
      "slide_number": 2,
      "type": "content",
      "title": "为什么选择 Nova？",
      "key_points": ["10 倍更快的处理速度", "类人理解能力", "企业级安全"],
      "visual_description": "相同的紫青渐变背景。左侧：浮动磨砂玻璃卡片带标题 '为什么选择 Nova？'（粗体白色），下方三个要点带微妙玻璃药丸标签。右侧：抽象 3D 神经网络可视化，如带柔和光晕的互联玻璃节点。浮动半透明几何形状（二十面体、圆环）增添深度。与前一张幻灯片一致的毛玻璃美学。"
    },
    {
      "slide_number": 3,
      "type": "content",
      "title": "工作原理",
      "key_points": ["自然语言输入", "多模态处理", "即时洞察"],
      "visual_description": "与前几张幻灯片一致的渐变背景。中央构图：三个带角度叠放的磨砂玻璃卡片展示工作流程步骤，由柔和发光线条连接。每张卡片带抽象图标。浮动玻璃球体和光粒子围绕构图。顶部标题 '工作原理'（粗体白色）。通过卡片分层和透明度营造深度。"
    },
    {
      "slide_number": 4,
      "type": "content",
      "title": "为规模而生",
      "key_points": ["100 万+ 并发用户", "99.99% 正常运行时间", "全球基础设施"],
      "visual_description": "相同的渐变背景。不对称布局：右侧带大磨砂玻璃面板，指标以粗体排版显示。左侧：抽象 3D 玻璃面板地球仪和连接线，代表全球规模。浮动数据可视化元素如带数字的小玻璃卡片。柔和环境光晕贯穿。高级科技美学。"
    },
    {
      "slide_number": 5,
      "type": "conclusion",
      "title": "未来从现在开始",
      "subtitle": "加入候补名单",
      "visual_description": "戏剧性结尾幻灯片。渐变背景带略微增加的活力。中央磨砂玻璃卡片带粗体标题 '未来从现在开始' 和行动号召副标题。卡片后方：柔和光线爆发和浮动玻璃粒子营造庆祝效果。多层玻璃形状营造深度。在保持风格一致性的同时最具视觉冲击力的幻灯片。"
    }
  ]
}
```

### 步骤 2：读取图像生成技能

读取 `/mnt/skills/public/image-generation/SKILL.md` 了解如何生成图像。

### 步骤 3：使用引用链按顺序生成幻灯片图像

**幻灯片 1 - 标题（建立视觉语言）：**

创建 `/mnt/user-data/workspace/nova-slide-01.json`：
```json
{
  "prompt": "超高级毛玻璃设计演示标题幻灯片。背景：从深紫色（#667eea）经品红（#f093fb）到青色（#00d4ff）的平滑流动渐变，柔和而充满活力。中央：带强背景模糊效果的大型磨砂玻璃面板，32px 圆角，包含粗体白色无衬线标题 'Nova AI 发布'（72pt，SF Pro Display 风格，font-weight 700）带微妙文字阴影，下方较轻权重副标题。玻璃面板带微妙白色边框（1px rgba 255,255,255,0.25）和柔和紫色调投影。围绕卡片浮动：带折射的 3D 玻璃球体、半透明几何形状（二十面体、抽象斑点），营造深度和维度。玻璃面板后散发柔和发光。小浮动光粒子。苹果 visionOS UI 美学。专业演示幻灯片，16:9 宽高比。超现代、高级科技产品发布感。",
  "style": "毛玻璃、visionOS 美学、苹果 Vision Pro UI 风格、高级科技、2024 设计趋势",
  "composition": "居中玻璃卡片作为焦点，浮动 3D 元素在边缘营造深度，40% 留白，清晰的视觉层次",
  "lighting": "来自渐变的柔和环境光晕、玻璃元素的光线折射、3D 形状的微妙边缘光",
  "color_palette": "紫色渐变 #667eea、品红 #f093fb、青色 #00d4ff、磨砂白 rgba(255,255,255,0.15)、纯白文本 #ffffff",
  "effects": "玻璃面板背景模糊、带色调的柔和投影、光线折射、玻璃上的微妙噪点纹理、浮动粒子"
}
```

```bash
python /mnt/skills/public/image-generation/scripts/generate.py \
  --prompt-file /mnt/user-data/workspace/nova-slide-01.json \
  --output-file /mnt/user-data/outputs/nova-slide-01.jpg \
  --aspect-ratio 16:9
```

**幻灯片 2 - 内容（必须引用幻灯片 1 以保持一致性）：**

创建 `/mnt/user-data/workspace/nova-slide-02.json`：
```json
{
  "prompt": "延续参考图像 EXACT 视觉风格的演示幻灯片。相同紫青渐变背景、相同毛玻璃美学、相同排版风格。左侧：带背景模糊的磨砂玻璃卡片包含标题 '为什么选择 Nova？'（粗体白色，匹配参考字体风格），下方三个功能点作为微妙玻璃药丸标签。右侧：抽象 3D 神经网络可视化，由带柔和青色光晕的互联玻璃节点组成，悬浮在空间中。浮动半透明几何形状（匹配参考风格）增添深度。磨砂玻璃处理完全相同：白色边框、紫色调阴影、相同模糊强度。关键：此幻灯片必须看起来与参考图像属于同一演示文稿 - 相同颜色、相同玻璃处理、相同整体美学。",
  "style": "完全匹配参考 - 毛玻璃、visionOS 美学、相同视觉语言",
  "composition": "不对称分割：玻璃卡片左侧（40%）、3D 可视化右侧（40%）、元素间留白",
  "color_palette": "完全匹配参考：紫色 #667eea、青色 #00d4ff 渐变、相同磨砂白处理、相同文本白色",
  "consistency_note": "关键：必须在视觉上与参考图像风格相同。相同渐变颜色、相同玻璃模糊强度、相同阴影处理、相同排版权重和风格。观看者应立即认出这是同一演示文稿。"
}
```

```bash
python /mnt/skills/public/image-generation/scripts/generate.py \
  --prompt-file /mnt/user-data/workspace/nova-slide-02.json \
  --reference-images /mnt/user-data/outputs/nova-slide-01.jpg \
  --output-file /mnt/user-data/outputs/nova-slide-02.jpg \
  --aspect-ratio 16:9
```

**幻灯片 3-5：继续相同模式，每张引用前一张幻灯片**

后续幻灯片的关键一致性规则：
- 始终在提示中包含"延续参考图像 EXACT 视觉风格"
- 指定"相同渐变背景""相同玻璃处理""相同排版"
- 包含强调风格匹配的 `consistency_note` 字段
- 引用紧邻的前一张幻灯片图像

### 步骤 4：组合最终 PPT

```bash
python /mnt/skills/public/ppt-generation/scripts/generate.py \
  --plan-file /mnt/user-data/workspace/nova-plan.json \
  --slide-images /mnt/user-data/outputs/nova-slide-01.jpg /mnt/user-data/outputs/nova-slide-02.jpg /mnt/user-data/outputs/nova-slide-03.jpg /mnt/user-data/outputs/nova-slide-04.jpg /mnt/user-data/outputs/nova-slide-05.jpg \
  --output-file /mnt/user-data/outputs/nova-presentation.pptx
```

## 特定风格指南

### 毛玻璃风格（推荐 - 最现代前卫）
```json
{
  "style": "glassmorphism",
  "style_guidelines": {
    "color_palette": "充满活力的渐变背景（紫色 #667eea 到粉色 #f093fb，或青色 #4facfe 到蓝色 #00f2fe），磨砂白面板 20% 不透明度，在渐变上突出的强调色",
    "typography": "SF Pro Display 或 Inter 字体风格，粗体 600-700 权重标题，干净 400 权重正文，白色文本带微妙投影以在玻璃上可读",
    "imagery": "悬浮在空间中的抽象 3D 形状、柔和模糊球体、玻璃材质的几何基元、通过重叠半透明层营造深度",
    "layout": "浮动卡片面板带背景模糊效果、充足内边距（48-64px）、圆角（24-32px 半径）、微妙阴影的层次深度",
    "effects": "磨砂玻璃模糊（backdrop-filter: blur 20px）、微妙白色边框（1px rgba 255,255,255,0.2）、面板后柔和光晕、带投影的浮动元素",
    "visual_language": "苹果 Vision Pro UI 风格的高级科技美学、通过透明度的深度、光线通过玻璃表面折射"
  }
}
```

### 暗色高级风格
```json
{
  "style": "dark-premium",
  "style_guidelines": {
    "color_palette": "深黑底色（#0a0a0a 到 #121212）、发光强调色（电光蓝 #00d4ff、霓虹紫 #bf5af2 或金色 #ffd700）、营造深度的微妙灰色渐变（#1a1a1a 到 #0a0a0a）",
    "typography": "优雅无衬线体（Neue Haas Grotesk 或 Suisse Int'l 风格）、戏剧性大小对比（72pt+ 标题、18pt 正文）、字间距 -0.02em 标题、纯白（#ffffff）文本",
    "imagery": "戏剧性工作室灯光、边缘光和发光、电影级产品拍摄、抽象光线轨迹、高级材质纹理（拉丝金属、磨砂表面）",
    "layout": "充足留白（60%+）、不对称平衡、内容锚定到网格但留有呼吸空间、每张幻灯片单一焦点",
    "effects": "关键元素后的微妙环境光晕、光线泛光效果、颗粒纹理叠加（2-3% 不透明度）、边缘暗角",
    "visual_language": "奢华科技品牌美学（Bang & Olufsen、Porsche Design）、通过克制展现精致、每个元素都经过深思熟虑"
  }
}
```

### 渐变现代风格
```json
{
  "style": "gradient-modern",
  "style_guidelines": {
    "color_palette": "大胆网格渐变（Stripe/Linear 风格：紫-粉-橙 #7c3aed→#ec4899→#f97316，或冷色调：青-蓝-紫 #06b6d4→#3b82f6→#8b5cf6），根据背景强度使用白色或深色文本",
    "typography": "现代几何无衬线体（Satoshi、General Sans 或 Clash Display 风格）、可变字重、超大粗体标题（80pt+）、舒适的正文（20pt）",
    "imagery": "抽象流体形状、变形渐变、3D 渲染抽象物体、柔和有机形态、浮动几何基元",
    "layout": "动态不对称构图、带混合模式的重叠元素、文本与渐变流融合、全出血背景",
    "effects": "平滑渐变过渡、微妙噪点纹理（3-5% 营造深度）、带色调的柔和投影匹配渐变、暗示运动的运动模糊",
    "visual_language": "当代 SaaS 美学（Stripe、Linear、Vercel）、活力而专业、前瞻科技感"
  }
}
```

### 新粗野主义风格
```json
{
  "style": "neo-brutalist",
  "style_guidelines": {
    "color_palette": "高对比度原色：纯黑、纯白，带大胆强调（热粉 #ff0080、电光黄 #ffff00 或原始红 #ff0000），可选：孟菲斯风格柔和色作为次要色",
    "typography": "超粗压缩字体（Impact、Druk 或 Bebas Neue 风格）、大写标题、极端大小对比、刻意紧凑或重叠的字间距",
    "imagery": "原始未过滤摄影、刻意的视觉噪点、半色调图案、剪贴画拼贴美学、手绘元素、贴纸和印章",
    "layout": "破碎网格、重叠元素、粗黑边框（4-8px）、可见结构、反留白（密集但有序的混乱）",
    "effects": "硬阴影（无模糊、偏移 8-12px）、像素化强调、扫描线、CRT 屏幕效果、刻意的"错误"",
    "visual_language": "反企业叛逆、DIY 杂志美学与数字化结合、原始真实性、通过大胆令人难忘"
  }
}
```

### 3D 等距风格
```json
{
  "style": "3d-isometric",
  "style_guidelines": {
    "color_palette": "柔和当代调色板：柔和紫色（#8b5cf6）、蓝绿色（#14b8a6）、温暖珊瑚色（#fb7185），奶油色或浅灰背景（#fafafa），元素间饱和度一致",
    "typography": "友好几何无衬线体（Circular、Gilroy 或 Quicksand 风格）、中等字重标题、出色的可读性、舒适的 24pt 正文",
    "imagery": "干净的等距 3D 插图、一致的 30° 等距角度、柔和粘土渲染美学、浮动平台和设备、可爱简化物体",
    "layout": "中央等距场景作为主体、文本围绕 3D 元素平衡、清晰的视觉层次、舒适的边距（64px+）",
    "effects": "柔和投影（20px 模糊、30% 不透明度）、3D 物体的环境光遮蔽、表面的微妙渐变、一致光源（左上）",
    "visual_language": "友好的科技插图（Slack、Notion、Asana 风格）、亲和的复杂性、通过简化实现清晰"
  }
}
```

### 编辑风格
```json
{
  "style": "editorial",
  "style_guidelines": {
    "color_palette": "精致中性色：米白（#f5f5f0）、炭灰（#2d2d2d），带单一强调色（酒红 #7c2d12、森林绿 #14532d 或海军蓝 #1e3a5f），偶尔全彩摄影",
    "typography": "精致衬线体用于标题（Playfair Display、Freight 或 Editorial New 风格）、干净无衬线体用于正文（Söhne、Graphik）、戏剧性大小层次（96pt 标题、16pt 正文）、宽松行高 1.6",
    "imagery": "杂志品质摄影、戏剧性裁剪、全出血图像、带刻意留白的肖像、编辑灯光（Vogue、Bloomberg Businessweek 风格）",
    "layout": "精致网格系统（12 列）、刻意不对称、引用作为设计元素、文本环绕图像、优雅边距",
    "effects": "最小效果 - 让摄影和排版闪耀、微妙图像处理（轻微去饱和、胶片颗粒）、优雅边框和标尺",
    "visual_language": "高端杂志美学、知识精致、通过设计克制提升内容"
  }
}
```

### 极简瑞士风格
```json
{
  "style": "minimal-swiss",
  "style_guidelines": {
    "color_palette": "纯白（#ffffff）或米白（#fafaf9）背景、纯黑（#000000）文本、单一醒目强调（瑞士红 #ff0000、克莱因蓝 #002fa7 或信号黄 #ffcc00）",
    "typography": "Helvetica Neue 或 Aktiv Grotesk，严格字号比例（12/16/24/48/96）、中等字重正文、仅强调时加粗、左对齐不齐右",
    "imagery": "客观摄影、几何形状、干净图标、数学精度、作为构图元素的刻意留白",
    "layout": "严格网格遵循（基线网格精神可见）、模块化构图、充足留白（40%+）、内容对齐到隐形网格线",
    "effects": "无 - 形式纯粹、无阴影、无渐变、无装饰元素、偶尔单一细线标尺",
    "visual_language": "国际排版风格、形式追随功能、永恒现代主义、Dieter Rams 风格克制"
  }
}
```

### 主题演讲风格（苹果风格）
```json
{
  "style": "keynote",
  "style_guidelines": {
    "color_palette": "深黑（#000000 到 #1d1d1f）、纯白文本、标志性蓝（#0071e3）或渐变强调（创意用紫粉、科技用蓝青）",
    "typography": "San Francisco Pro Display，极端字重对比（粗体 80pt+ 标题、轻体 24pt 正文）、标题负字间距（-0.03em）、光学对齐",
    "imagery": "电影级摄影、浅景深、戏剧性灯光（边缘光、聚光灯）、带反射的产品主体拍摄、全出血图像",
    "layout": "最大留白、每张幻灯片一个强有力的图像或声明、居中或戏剧性偏移内容、无杂乱",
    "effects": "微妙渐变叠加、关键元素的光线泛光和发光、表面反射、平滑渐变背景",
    "visual_language": "苹果 WWDC 主题演讲美学、通过简约展现自信、每个像素都经过考量、戏剧性呈现"
  }
}
```

## 输出处理

生成后：

- PPTX 文件保存在 `/mnt/user-data/outputs/`
- 使用 `present_files` 工具与用户分享生成的演示文稿
- 如果需要，也分享单独的幻灯片图像
- 提供演示文稿的简要描述
- 如果需要，提供迭代或重新生成特定幻灯片的选项

## 注意事项

### 关键质量指南

**专业结果的提示工程：**
- 始终使用英文撰写图像提示，无论用户使用什么语言
- 对视觉细节要极其具体 - 模糊的提示会产生通用结果
- 包含精确的十六进制颜色代码（例如 #667eea 而非"紫色"）
- 指定排版细节：字重（400/700）、大小层次、字间距
- 精确描述效果："背景模糊 20px""投影 8px 模糊 30% 不透明度"
- 参考真实设计系统："visionOS 美学""Stripe 网站风格""Bloomberg Businessweek 布局"

**视觉一致性（最重要）：**
- **按顺序生成幻灯片** - 每张幻灯片必须引用前一张
- 第一张幻灯片至关重要 - 它为整个演示文稿建立视觉语言
- 在每张后续幻灯片提示中明确说明："延续参考图像 EXACT 视觉风格"
- 在提示中强调使用 SAME、EXACT、MATCH 关键词以强制一致性
- 在幻灯片 1 之后的每个 JSON 提示中包含 `consistency_note` 字段
- 如果某张幻灯片看起来不一致，用更强的参考强调重新生成

**现代美学的设计原则：**
- 拥抱留白 - 40-60% 空白空间营造高级感
- 限制每张幻灯片的元素 - 一个焦点，一个信息
- 通过分层营造深度（阴影、透明度、z 轴深度）
- 排版层次：巨大标题（72pt+）、舒适正文（18-24pt）
- 颜色克制：一个主色调、最多 1-2 个强调色

**常见错误避免：**
- ❌ 通用提示如"专业幻灯片" - 要具体
- ❌ 每张幻灯片太多元素/文本 - 杂乱 = 不专业
- ❌ 幻灯片间颜色不一致 - 始终引用前一张幻灯片
- ❌ 跳过参考图像参数 - 这会破坏视觉一致性
- ❌ 在同一演示文稿中使用不同的设计风格
- ❌ 并行生成幻灯片 - 幻灯片必须按顺序逐一生成（幻灯片 1 → 2 → 3 ...），绝不能同时生成

**不同场景的推荐风格：**
- 科技产品发布 → `glassmorphism` 或 `gradient-modern`
- 奢华/高级品牌 → `dark-premium` 或 `editorial`
- 初创公司推介 → `gradient-modern` 或 `minimal-swiss`
- 高管演示 → `dark-premium` 或 `keynote`
- 创意机构 → `neo-brutalist` 或 `gradient-modern`
- 数据/分析 → `minimal-swiss` 或 `3d-isometric`