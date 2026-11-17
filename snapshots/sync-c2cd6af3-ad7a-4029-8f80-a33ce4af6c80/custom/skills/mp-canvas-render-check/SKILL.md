---
name: 'mp-canvas-render-check'
description: '在没有微信开发者工具的环境里，用无头 Chrome 渲染小程序 canvas 绘制代码（分享海报 / 分享图 / 图表 / 水印），产出 PNG 供肉眼核对与迭代；同套手段也可压缩/生成图片资源（无 PIL / sharp 时降采样大图以控制主包体积）。当需要校验或调优小程序 canvas 2d 绘制结果、对比设计稿、排查排版问题、压缩小程序图片资源时使用。触发词：小程序 canvas、海报渲染、渲染预览、无开发者工具验证、canvas 截图核对、poster render check、图片压缩、降采样、主包体积。'
---

# 小程序 canvas 渲染校验（无头 Chrome）

## 什么时候用

- 写了 / 改了小程序里用 `canvas 2d` 绘制的代码（海报、分享图、报表、合成图），但不想（或不能）开微信开发者工具
- 需要把渲染结果与设计稿逐一比对、反复微调
- 小程序 canvas 2d 的 API 与 **HTML Canvas 2D 基本一致**，所以同一份绘制代码可以在浏览器里跑

## 核心思路

**不重写绘制逻辑**，而是把小程序项目里真实的 CommonJS 模块原样内嵌进一个自包含 HTML，在浏览器里加载并执行，再截图。

```
真实源码（utils/poster.js 等）
   ↓ 内嵌为 <script type="text/plain" id="src-xxx">
自包含 render.html（无 fetch → file:// 直接可用）
   ↓ new Function('module','exports','require', code)  手动实现 CommonJS
mods 映射表 → 被校验模块拿到自己的依赖
   ↓
canvas 2d 真实上下文 → 调用真实 layout()/draw()
   ↓ chrome --headless --screenshot=<绝对路径>
PNG（用 Read 工具看）+ <pre id="diag"> 里的数值诊断
```

## 用法

```bash
node scripts/render-check.js <config.json>
```

`config.json`：

```json
{
  "modules": {
    "../data/config.js": "D:/proj/miniAPP/data/config.js",
    "./share.js":        "D:/proj/miniAPP/utils/share.js",
    "./poster.js":       "D:/proj/miniAPP/utils/poster.js"
  },
  "sketch": "sketch.js",
  "out": "out.png",
  "viewport": { "width": 1080, "height": 1840 },
  "chrome": "C:/Program Files/Google/Chrome/Application/chrome.exe"
}
```

- `modules` 的 key 必须与源码里 `require()` 的字符串**逐字一致**
- `viewport` **默认会被忽略**（截图自动按画布尺寸取，见坑 11）；想固定窗口尺寸（例如故意要连诊断条一起截）就加 `"viewportMode": "fixed"`
- `sketch.js` 见 `templates/sketch.example.js`：一个普通脚本（不是 CommonJS），
  必须定义 `async function render({ mods, canvas, ctx, log })`，返回值约定为最终 canvas 高度
- 脚本会同时跑一次 `--dump-dom`，把 `log()` 的内容原样打印到终端 → **把行高、总高、坐标等数值 log 出来**，不用靠眼睛量
- 产物：`<config 所在目录>/render.html`、`out.png`

## 必须注意的坑

1. **不能用 `fetch` 读源码**：`file://` 下跨源被拦，所以必须把源码内嵌进 HTML（脚本已做）。
   想省事改用本地 http server 也行，但内嵌更省依赖。
2. **不能用 `<script src>` 直接加载 CommonJS 源码**：多个模块的顶层 `const` 会落在同一个全局词法作用域里，
   同名即 `SyntaxError: Identifier 'X' has already been declared`。用 `new Function(...)` 包一层才是隔离的。
3. **`chrome --screenshot` 的输出路径要用绝对 Windows 路径**（`C:\...\out.png`），
   配合 Git Bash 时相对路径常常静默不产出文件。
4. 用 `--headless`（老模式）on Windows 更稳；新无头模式下 `--screenshot` 有概率不出文件。
5. 页面里的异步逻辑（图片加载等）要给 `--virtual-time-budget=8000` 留时间，否则截到半成品。
6. 要 1:1 看设计稿像素，加 `--force-device-scale-factor=1 --hide-scrollbars`，
   并在 sketch 里把 `canvas.style.width/height` 设成 `canvas.width/height`。
7. **本地图片可以正常加载**（`<img src>` / `canvas.createImage()` 换成 `new Image()`），
   相对路径以 render.html 所在目录为基准 —— 需要用到项目里的 logo 时先 `cp` 到该目录。
8. 字体：浏览器用系统 `sans-serif`，小程序真机是 PingFang SC / Noto Sans SC，
   **度量会有几个像素级差异**（换行位置、行数可能差 1 行）。用数值诊断确认公式正确即可，别死磕像素。
9. **沙箱里的模块副本必须与仓库源码逐字节一致**：`config.json` 的 `modules` 指向真实路径，
   但渲染工作在沙箱目录，第二次迭代时常去改沙箱里那份副本 → 改完以为验证过了，其实仓库没变
   （或反之：改了仓库忘了同步沙箱 → 一路在验证旧代码）。
   **每次渲染前先比对一次**，成本极低：
   ```bash
   node -e "const fs=require('fs');const a=fs.statSync('<沙箱>/poster.js').size,b=fs.statSync('<仓库>/miniAPP/utils/poster.js').size;console.log(a,b,a===b?'SAME':'STALE!')"
   ```
   视觉上也容易骗人：只差一行像素的旧代码截出来的图**看不出差别**，必须靠这个断言兜底。
10. **截图里若有「预期不该出现」的图形，先怀疑自己在看旧图**，不要急着改绘制代码。
   典型误判：看到码外有虚线框 → 以为画了占位框，其实渲染用的是真实码图
   （虚线框是圆码最外圈散点被缩放后看成的）—— 用第 9 条的一致性断言即可排除。
11. **截图尺寸要等于画布尺寸，否则会把页面上的其它元素一起截进来**。
   `--window-size` 比画布高时，画布下方的诊断 `<pre>`（脚本里自带灰底 `#9aa0a6`）会占满剩余高度，
   产出的就**不是一张干净的海报** —— 改了画布高度而忘记同步 `config.viewport` 时必然发生，
   而且这个灰条很像「设计的一部分」，容易被忽略就发布出去。
   `render-check.js` 现在会先从 `--dump-dom` 的诊断里解析 `canvas backing store = W x H`，
   截图窗口尺寸直接取画布尺寸；只有显式写 `"viewportMode": "fixed"` 才用 `config.viewport`。

## 别用肉眼量留白，用像素量

海报动辄 1920 宽，在聊天窗口里按 1/2.3 显示 —— **10px 和 20px 的内边距在缩放图上看不出差别**，
也分辨不出「改生效了」和「跑的是旧图」。需要确认内边距 / 间距 / 居中时，直接用像素问答案：

```bash
node scripts/measure-png.js <img.png> <x0> <y0> <x1> <y1>
# 区域填「单元格/卡片」的矩形，输出该区域内非背景像素的包围盒 + 到四边的距离
node scripts/measure-png.js poster.png 148 596 491 729
#   → 左留白 21px   上留白 34px   右留白 101px   下留白 33px
```

- 纯 Node（`zlib` + 手写反滤波），**无需 PIL / sharp / node-canvas**；支持 8-bit 的 colorType 0/2/3/4/6
- 背景色从区域四角自动估计，前景判定默认 `lum < 150`（深色文字），可用 `--ink` 调
- 实测 ≈「代码里的数值 + 1px」（抗锯齿的边缘像素被算作前景），所以对得上就是生效了
- 这个手段曾经直接推翻过需求前提：用户说「内边距改成 10」，量出来发现**原本就是 10**，
  真正的问题是缩放后观感贴边 → 改成 20 才解决

### 量「行距 / 行间净空」—— `measure-rowgap.js`

排版类需求（「每行间距太紧了」「行距拉开一点」）**不能靠落档参数自证**：`lh` 是行距基线，
观感松紧取决于 `lh − 墨迹高`。中文墨迹高约 `0.87 × fs`，所以 `lh = 1.13 × fs` 时净空只有 ~6px，
字几乎贴在一起 —— 光看 `fs/lh` 两个数字看不出来。直接扫像素：

```bash
node scripts/measure-rowgap.js <img.png> <x0> <y0> <x1> <y1>
# 区域取「一个格子里的一列文字」，脚本会按行扫描前景像素段
node scripts/measure-rowgap.js poster.png 854 916 1167 1154
#   行 1  y  934~ 963  墨迹高 30px
#   行 2  y  977~1006  墨迹高 30px   与上一行净空 13px
#   ...
#   墨迹行数 5 | 平均墨迹高 30.0px | 行距(基线) 43.0px | 行间净空 13.0px (min 13 / max 13)
```

- 输出里的 `行距(基线)` **应等于排版落档的 lh** —— 不等就说明量错了区域或落档不是你以为的那档
- `行间净空` 才是观感指标；`min` 比平均值更有用（汉字有升降部，逐行会浮动几像素）
- 同一个脚本可以拿改版前后的两张图跑对比，**「字号没动、净空翻倍」这种结论只有像素能证明**
- 同一目录还有 `measure-png.js` 可量「文字块到格线的留白」，两者配合能把一个格子的排版拆干净

## 导出体积必须实测，别信 sketch 里的 toDataURL 估算

sketch 常用 `canvas.toDataURL('image/png').length * 0.75 / 1024` 估体积。**这个估算会高报约 35%**
（同样是 PNG，`toDataURL` 出来的字节数比 `--screenshot` 落盘的实际文件大），
实测同一张图：估算 546KB / 实际 404.5KB。把估算值写进文档会让人误判分享耗时。

要真实体积，就让画布按倍率放大后**截图落盘**再读文件大小：

```js
// sketch-2x.js：把 backing store 设成逻辑尺寸的 2 倍，layout 仍按逻辑坐标算
const SCALE = 2;
canvas.width = P.width * SCALE;
canvas.height = P.height * SCALE;
ctx.setTransform(1, 0, 0, 1, 0, 0);
ctx.scale(SCALE, SCALE);      // 之后 draw() 不用改
return canvas.height;         // render-check 按返回值定截图窗口
```

- 派生 2x 版 sketch 时**别复制粘贴整份文件**（WEEK 之类的测试数据会立刻漂移）。
  用脚本做文本替换生成，改主 sketch 后重新派生一次即可
- sketch 是在 `new Function(src + ';return render;')()` 里跑的，**作用域里没有 `require`** ——
  不能 `require('./week.js')` 抽公共数据，只能整份派生
- 3840×2744 的截图 Chrome 完全扛得住（上限 16384），不必担心窗口尺寸
- 另一条捷径：微信聊天图最终会被压到长边 ≤1280，**「存相册后的真实观感」看的是 1280 版本的码像素**，
  而不是导出倍率 —— 所以倍率主要影响体积和相册放大清晰度，不影响识别率

## 附：顺手用它压缩 / 生成图片资源（无 PIL、无 sharp）

同一套「无头 Chrome + canvas」手段还能解决小程序主包体积问题。两个脚本：

### 1) 降采样 —— `img-downscale.js`

```bash
node scripts/img-downscale.js <src图片> <输出路径> [目标边长=360] [png|jpeg]
# node scripts/img-downscale.js src/gh_xxx_1280.jpg miniAPP/assets/qrcode/mini-code.png 360 png
```

- 脚本按 **2 倍逐级降采样**（1280→640→360）；一次性缩放会有明显锯齿
- 结果经 `toDataURL` 写进 DOM，再 `--dump-dom` 取回 base64 落盘 —— 不能靠 `--screenshot`
  （截图会被 window 尺寸/缩放取整污染）
- 如果使用方是「扫得出来的码」类素材，输出用 **PNG**（无损），不要为了再省几十 KB 用 JPEG

### 2) 颜色量化到 PNG-8 —— `img-quantize.js`（省得更多，画质反而更干净）

canvas 只能导出 PNG-24。**纯色块 / 码图 / 插画**这类颜色数极少的图，量化到 128 色体积能降一个数量级，
而且顺带抹掉源 JPEG 的压缩伪影（对识码是净收益）。

```bash
node scripts/img-quantize.js <src图片> <输出.png> [目标边长=900] [色数=128] [近白阈值=238]
# node scripts/img-quantize.js src/gh_xxx_1280.jpg miniAPP/assets/qrcode/mini-code.png 900 128 238
# → 366 KB → 64.8 KB（省 82%），1280→900 分辨率还提升了
```

- 浏览器内做「中位切分量化 → 手写 PNG-8（IHDR/PLTE/IDAT/IEND）→
  `CompressionStream('deflate')` 压 IDAT → 自实现 CRC32」
- **近白阈值 > 0 时会把近白像素强制成纯白**。二维码/小程序码靠四周安静区识别，
  量化容易把白背景和抗锯齿色混成一坨浅灰，必须强行保住纯白
- 实测对比（同一张 900px 圆码）：PNG-24 = 457KB / 128 色 = 47KB / 64 色 = 32KB
- **注意**：Node 侧路径要用 Windows 风格（`C:/...`）；传 Git Bash 风格（`/c/...`）会被解析成
  相对路径，文件落到奇怪的地方

### ⚠️ 目标尺寸别只看「canvas 里的渲染尺寸」

对**要被人长按识别的码图**，尺寸要按「码在整张图里的**占比**」反推 —— 因为
**微信会把聊天窗口发送的图片压到长边 1280 以内**（除非用户勾选「发送原图」），
代码里画得再大也没用：

```
压缩后码边长 = 1280 × (码宽 / 画布宽)
目标：≥ 300px 最稳；< 200px 基本识别不了
→ 反推：码占画布宽 ≥ 23%
```

真实案例：旧版码占宽 14.8%（1080 宽画布里的 160px）→ 压缩后仅 190px、点阵有效直径约 165px
→ **长按无反应**；改成占宽 24%（1920 宽画布里的 460px）→ 压缩后 307px ✓。
同时源图必须够清晰（圆码靠**三个同心圆定位点**识别，降采样过度会糊掉），
实测 360px 源图不够、900px 够。

**排查这类问题的方法**（本 skill 的 render-check 之外，补一个压缩模拟）：
把渲染出的整图用 canvas 缩到 1280 宽 + `toDataURL('image/jpeg', 0.8)`，
回读后再裁出码区域、`imageSmoothingEnabled=false` 放大 2 倍 —— 就是接收方看到的画质。

## 浏览器路径（Windows 常见位置）

```
C:/Program Files/Google/Chrome/Application/chrome.exe
C:/Program Files (x86)/Microsoft/Edge/Application/msedge.exe
```

## 参考案例

`D:\QoneLab\my_Code\shimie` 的分享海报：`AI任务/海报/分享海报模板规范.md`（设计规范）
+ `miniAPP/utils/poster.js`（真实绘制代码），用本方法渲染出
`AI任务/海报/分享海报-小程序渲染预览.png` 与 `分享海报-渲染预览-数据量最大周.png`（压力用例），
与 `分享海报模板-标注稿.png` 逐项比对通过。

现版式为 **1920 × 1372**（全宽菜谱表 1808×812 + 右上角小程序码 360，码占画布宽 18.8%）。
两个用例都跑到「零缩字、零折行」，靠 sketch 里的 `log()` 打出实际落档的
`fs/lh/rp` + 折行次数 + 总高，一目了然 —— **排版类改动一定要 log 出落档档位**，
否则「看起来对」和「其实降级到了小字号」肉眼分不出。

> 一次典型的「改版式但零回归」：该海报表体可用高曾长期恒为 608px，
> 画布从 1920×1620 → 1364 → 1260 变了两次，表格排版落点却完全没变 ——
> **只要诊断里的「表体可用高」不变，跑一次渲染就能确认没回归**，不必逐个用例肉眼比。
> 这正是把关键约束 log 出来的价值。

> 反过来也成立，而且是本项目里最有价值的一次推导：用户反馈「菜名每行间距太紧」，
> 排查发现**根因不是阶梯参数，而是表体高度**——表体 608 装最坏周（Σ16 行）时，
> 行距的数学上限只有 ~38px（净空 8px），怎么调阶梯都拉不开。于是加高画布把表体提到 720。
> **当「调参数怎么都不对」时，先回去算约束够不够，而不是继续调参数。**