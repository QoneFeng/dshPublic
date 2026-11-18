---
name: 'playwright-workflow'
description: '浏览器 capability skill，负责通用浏览器、Playwright runtime、自定义 Node 探针、session 与 storageState 能力。'
---

# Playwright Workflow

使用这个技能处理通用浏览器任务，包括 `playwright-cli` 封装命令和自定义 Node Playwright 探针。

它负责：

1. 通用浏览器入口脚本
2. session / storageState 运行时规则
3. 运行时清理脚本
4. 新技能共享的路径解析辅助
5. 共享 Playwright runtime 的安装、检查和复用
6. 默认关闭 WebM 录像；仅在准备上传 PMS 代表性证据或用户明确要求时，按同一路径复跑并录制
7. 统一真实页面验收的产物目录、日志、结果分类、有限重试和录像校验基线

## 默认原则

- 第一次使用或发现 `require('playwright')` / `playwright-cli` 不可用时，先运行 `scripts/browser_ensure_playwright_runtime.js` 安装共享 runtime；不要在每次任务里临时 `npm install playwright`
- 共享 runtime 只负责 Node 依赖和 `playwright-cli`；如果本机有系统 Chrome，优先使用 `channel: 'chrome'` 或 `--browser=chrome`；Linux 没有系统 Chrome 但已安装 Playwright bundled Chromium 时，公共 CLI 脚本才默认改用 `--browser=chromium`
- 自定义 Playwright 探针默认 `chromium.launch({ channel: 'chrome', headless: true })`；需要显示浏览器时只改 `headless: false`
- 复杂页面调试、3D 渲染验证、运行态 API 调用和需要落盘证据的流程，优先写 `.tmp/<task>/probe.js` 这类自定义 Node 探针；`playwright-cli` 更适合打开页面、保存登录态、简单 snapshot / run-code
- 先用当前 storageState 实际打开目标系统，验证是否已经处于可用登录态
- 只有确认落到登录页、认证失效页，或无法进入目标业务页时，才提醒用户登录
- 需要登录时，先打开浏览器，再让用户完成登录
- 默认按“小白用户第一次使用”组织提示语，一次只给一个动作
- 正式 storageState 与临时产物分目录保存，不要混放
- storageState 根目录可用 `PLAYWRIGHT_STORAGE_STATE_DIR` 覆盖；未设置时默认使用运行脚本时当前业务项目的 `.tmp`。3D 长效 token 保存在用户级目录，Playwright runtime 保存在用户级固定目录；项目 `.tmp` 只放同步出来的 `design-3d-<env>.json`、PMS/Gerrit storageState、测试模板和输出。
- 不要把 `launchPersistentContext`、`userDataDir`、`--user-data-dir` 或“profile 目录”作为常规登录态方案；它们会在项目或临时目录留下大量 profile 文件。只有 storageState 和长效凭证无法复用、且需要兼容真实用户 Chrome profile 时，才作为明确 fallback 使用
- 优先读取结构化快照，再读取截图或手动页面细节
- Bug 复现和修复的普通调试默认不录制 WebM，避免拖慢页面和 3D 渲染；准备上传 PMS 代表性证据或用户明确要求时，才在同一脚本上启用 `--record-video`，修复前展示错误事实，修复后按同一路径展示目标行为已恢复。录像不替代确定性断言、截图、console/network 日志和退出码
- 正式验收统一使用 `PASS` / `FAIL` / `BLOCKED`：目标断言闭环才是 `PASS`，稳定产品反证才是 `FAIL`，登录、动态模块、SDK、浏览器协议或测试数据前置无法完成时是 `BLOCKED`

## 依赖边界

- 本技能依赖：无

## 最小流程

1. 先确认共享 Playwright runtime 已安装；缺失时运行 `node scripts/browser_ensure_playwright_runtime.js`。
2. 再解析目标技能目录、storageState 路径、runtime 目录和当前系统可用浏览器。
3. 用当前 storageState 打开目标页，确认是否已进入业务页。
4. 如果未登录，打开 headed 浏览器并引导用户完成登录。
5. 登录后重新打开目标页，并再次确认已经不是登录页。
6. 然后再给上层技能提供 snapshot、click、screenshot、state-save 等能力。
7. 需要时把临时结果写入当前业务项目 `.tmp/<task>/`，把项目脚本可直接使用的 storageState 写回当前业务项目 `.tmp` 下的 JSON 文件。

## 自定义 Node 探针基线

当脚本需要直接 `require('playwright')`，不要在业务仓库安装依赖。优先由 `design_3d_prepare_workspace.js` 或模板脚本自动解析；手写脚本才临时把用户级共享 runtime 的 `node_modules` 放到 `NODE_PATH`：

```powershell
$env:NODE_PATH = node <playwright-workflow-scripts>\browser_ensure_playwright_runtime.js --print-node-path
node .tmp\<task>\probe.js
```

临时诊断脚本不要用 `node -` 加顶层 `await`，也不要在同一段 stdin 里混用 `require()` 和顶层 `await`。需要异步时写成 `.tmp/<task>/probe.cjs`，或用 CommonJS `async main()` 包裹后执行。

探针默认使用系统 Chrome 和 storageState：

```js
const { chromium } = require("playwright");

async function main() {
  const browser = await chromium.launch({
    channel: "chrome",
    headless: true,
    args: ["--ignore-certificate-errors"],
  });
  try {
    const context = await browser.newContext({
      storageState,
      ignoreHTTPSErrors: true,
    });
    const page = await context.newPage();
    await page.goto(targetUrl, { waitUntil: "domcontentloaded" });
  } finally {
    await browser.close();
  }
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

这里优先用本机系统 Chrome，不依赖 Playwright 下载的 Chromium / headless shell。Linux 如果没有系统 Chrome，先确认 Playwright bundled Chromium 已安装，再改为 `chromium.launch({ headless: true })` 或设置 `PLAYWRIGHT_EXECUTABLE_PATH` 指向已存在的 Chromium。只有系统 Chrome 和 bundled Chromium 都不可用，或任务必须复现 bundled browser 差异时，才考虑安装 Playwright 浏览器。

## 组合方式

- 作为浏览器 runtime、session、storageState 与路径能力的共享底座
- 供 `design-workflow`、`pms-workflow`、`gerrit-workflow` 直接复用
- 统一提供浏览器命令入口、运行时规则与产物清理能力

## 先读哪些文档

- 运行时 / 路径规则：
  [./references/runtime-path-rules.md](./references/runtime-path-rules.md)
- 浏览器 / 会话经验：
  [./references/browser-session-notes.md](./references/browser-session-notes.md)
- 验收产物、结果分类、有限重试和 WebM 校验：
  [./references/acceptance-artifacts.md](./references/acceptance-artifacts.md)