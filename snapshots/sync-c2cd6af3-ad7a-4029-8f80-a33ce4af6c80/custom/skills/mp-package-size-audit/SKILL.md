---
name: 'mp-package-size-audit'
description: '诊断并修复微信小程序主包/分包体积超限（上传报「源码包超出最大限制 source size … exceed max limit 2MB」）。用静态依赖闭包模拟开发者工具的「无依赖文件过滤」，算出真正进包的文件与体积、列出会被过滤的文件、以及会被误杀而必须写进 packOptions.include 的动态引用。触发词：主包超限、代码包超出最大限制、source size exceed max limit 2MB、包体积优化、ignoreUploadUnusedFiles、miniprogram_npm 太大、上传失败 代码包。'
---

# 小程序包体积体检与超限修复

## 什么时候用

- 上传体验版报 `Error: 源码包超出最大限制，source size 2297KB exceed max limit 2MB`
- 主包/分包体积告警、TDesign / Vant 等组件库导致包体膨胀
- 加了新功能后想确认包体积余量

## 一句话根因（先查这个）

主包超限 **九成不是业务代码大，而是 `project.config.json` 里的
`setting.ignoreUploadUnusedFiles` 为 `false`** —— 这会让整个 `miniprogram_npm`（组件库全量，
动辄 3~4MB）原样打进包。

典型症状：**本地预览一切正常，一上传就超限**。因为本地预览/真机调试走的是
`setting.ignoreDevUnusedFiles`（默认 `true`，就近写在 `project.private.config.json` 里），
上传走的才是 `ignoreUploadUnusedFiles`（默认 `false`）。两个开关只差一个词，
结果一个过滤、一个不过滤。

| 字段 | 作用范围 | 默认 |
|---|---|---|
| `setting.ignoreUploadUnusedFiles` | **上传**时过滤无依赖文件 | `false` ⚠️ |
| `setting.ignoreDevUnusedFiles` | 预览 / 真机调试 / 本地模拟器过滤 | `true` |

## 用法

```bash
node scripts/audit.js <小程序根目录>
node scripts/audit.js <小程序根目录> --json     # 给程序消费
```

输出四段：① 会进包的文件与体积 ② 无依赖文件（会被过滤） ③ 仅靠 JS 字符串引用的文件
④ `packOptions.include` 里是否指向了不存在的文件。最后给出「预计进包体积」与是否超 2MB。

> 校准技巧：改配置**之前**先跑一次，看第 ① + ② 的「项目文件总体积」是否≈报错里的 source size。
> 相等说明工具确实一个文件都没过滤（根因确认）；差很多则说明超限另有原因（图片资源过大、
> 分包配置错误等）。

## 修复步骤

### 1. 打开上传过滤

```json
// project.config.json
{ "setting": { "ignoreUploadUnusedFiles": true } }
```

### 2. 把「静态分析看不见的引用」写进 include

依赖分析只看 `require` / `usingComponents` / `@import` / wxml 里的静态 `src`。
**看不见**这些写法，一旦开了过滤就会被误杀 → 真机上图片加载失败：

- JS 里写死的路径常量，交给 `canvas.createImage()` / `wx.getImageInfo()` / `imageUrl`：

```js
const QR = '/assets/qrcode/mini-code.png';   // 依赖分析看不见
canvas.createImage().src = QR;
```

```json
{
  "packOptions": {
    "include": [
      { "type": "file", "value": "assets/qrcode/mini-code.png" },
      { "type": "file", "value": "assets/share/share-poster-5x4.jpg" }
    ]
  }
}
```

`include` 优先级高于 `ignore`，且**同时**作用于预览与上传。
`value` 以 `miniprogramRoot` 为根目录；`type` 可取 `file` / `folder` / `suffix` / `prefix` / `regexp` / `glob`。

### 3. 顺手清掉「会被误判为引用」的注释

JS 注释里如果写了完整的项目内路径（如 `// 原图 assets/logo/xxx_1280.jpg`），
某些版本的依赖分析会把注释里的字符串当成引用，**让本该被过滤掉的死文件继续留在包里**。
把注释里的文件名去掉后缀即可（写「1280 原图」而不是完整路径）。

### 4. 重新打开项目再上传

`packOptions` 的改动**需要关闭并重新打开项目**才生效（官方文档明确说明）。

## 必须注意的坑

1. **别用 `packOptions.ignore` 去硬编码一长串「未使用的组件目录」**。列 80 个目录既难维护，
   又埋雷：哪天要用那个组件，运行时才会静默报组件找不到。打开
   `ignoreUploadUnusedFiles` 让工具自动算，是官方且免维护的做法。
2. **组件库的字体图标若是远程 CDN**（如 TDesign 的 `https://tdesign.gtimg.com/icon/...`），
   过滤不会影响它，但要有「首次冷启动图标短暂缺失」的心理准备。
3. **`miniprogram_npm/<pkg>/.wechatide.ib.json`** 是开发者工具的构建信息，不是运行时代码，
   被过滤掉无所谓。
4. 主包 2MB 是**上传硬限制**，`setting.bigPackageSizeSupport`（把预览上限放宽到 4MB）
   救不了上传，别指望它。
5. 分包不能解决主包超限，除非把大依赖挪进分包并用**分包异步化**加载 —— 成本远高于打开过滤开关。

## 参考案例

`D:\QoneLab\my_Code\shimie\miniAPP`（TDesign + 4 个页面）：

- 现象：上传报 `source size 2297KB exceed max limit 2MB`，本地预览正常
- 根因：`ignoreUploadUnusedFiles: false` + `ignoreDevUnusedFiles: true`
- 体检：项目可打包文件 2214.8 KB ≈ 报错的 2297KB（确认零过滤）
- 修复：打开上传过滤 + `packOptions.include` 补 4 张 canvas/分享用图
  （`assets/logo/splash-logo.png`、`assets/qrcode/mini-code.png`、两张分享卡片封面）
  - 其中 splash-logo 当时尚有启动页 wxml 的静态引用、其实不会被过滤，仍显式登记 ——
    避免「哪天启动页改版，海报里的 logo 就凭空消失」这种隐式耦合
- 结果：预计进包 **700 KB**，其中 tdesign 仅保留 icon / button / tag / loading / common 样式