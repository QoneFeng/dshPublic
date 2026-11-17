---
name: 'agent-browser'
description: '用于人工智能代理的浏览器自动化命令行界面。当用户需要与网站进行交互时使用，这些交互包括浏览页面、填写表单、点击按钮、截图、提取数据、测试网页应用程序，或自动化任何浏览器任务。触发条件包括诸如"打开一个网站""填写一份表单""点击一个按钮""截图""从页面抓取数据""测试这个网页应用程序""登录到一个网站""自动化浏览器操作"等请求，或者任何需要通过编程实现网页交互的任务。'
---

# 使用 agent-browser 进行浏览器自动化

该命令行工具通过 CDP 直接使用 Chrome/Chromium。可通过 `npm i -g agent-browser`、`brew install agent-browser` 或 `cargo install agent-browser` 安装。运行 `agent-browser install` 下载 Chrome。已有的 Chrome、Brave、Playwright 和 Puppeteer 安装会自动被检测。运行 `agent-browser upgrade` 更新到最新版本。

## 核心工作流程

每个浏览器自动化都遵循以下模式：

1. **导航**: `agent-browser open <url>`
2. **快照**: `agent-browser snapshot -i`（获取元素引用，如 `@e1`、`@e2`）
3. **交互**: 使用引用进行点击、填写、选择
4. **重新快照**: 导航或 DOM 更改后，获取新的引用

```bash
agent-browser open https://example.com/form
agent-browser snapshot -i
# 输出: @e1 [input type="email"], @e2 [input type="password"], @e3 [button] "Submit"

agent-browser fill @e1 "user@example.com"
agent-browser fill @e2 "password123"
agent-browser click @e3
agent-browser wait --load networkidle
agent-browser snapshot -i  # 检查结果
```

## 命令链式调用

命令可以通过 `&&` 在单个 shell 调用中链接。浏览器通过后台守护进程在命令之间保持状态，因此链式调用是安全的，且比单独调用更高效。

```bash
# 在一次调用中链接 open + wait + snapshot
agent-browser open https://example.com && agent-browser wait --load networkidle && agent-browser snapshot -i

# 链接多个交互操作
agent-browser fill @e1 "user@example.com" && agent-browser fill @e2 "password123" && agent-browser click @e3

# 导航并捕获
agent-browser open https://example.com && agent-browser wait --load networkidle && agent-browser screenshot page.png
```

**何时使用链式调用：** 当不需要读取中间命令的输出即可继续时使用 `&&`（例如 open + wait + screenshot）。当需要先解析输出时（例如通过快照发现引用，然后使用这些引用进行交互），应分别运行命令。

## 处理认证

当自动化需要登录的网站时，选择适合的方式：

**方式 1：从用户浏览器导入认证（一次性任务最快）**

```bash
# 连接到用户正在运行的 Chrome（用户已登录）
agent-browser --auto-connect state save ./auth.json
# 使用该认证状态
agent-browser --state ./auth.json open https://app.example.com/dashboard
```

状态文件包含明文会话令牌——请添加到 `.gitignore` 并在不需要时删除。设置 `AGENT_BROWSER_ENCRYPTION_KEY` 可进行静态加密。

**方式 2：持久化配置文件（重复任务最简单）**

```bash
# 首次运行：手动或通过自动化登录
agent-browser --profile ~/.myapp open https://app.example.com/login
# ... 填写凭据，提交 ...

# 后续所有运行：已自动认证
agent-browser --profile ~/.myapp open https://app.example.com/dashboard
```

**方式 3：会话名称（自动保存/恢复 cookies + localStorage）**

```bash
agent-browser --session-name myapp open https://app.example.com/login
# ... 登录流程 ...
agent-browser close  # 状态自动保存

# 下次使用：状态自动恢复
agent-browser --session-name myapp open https://app.example.com/dashboard
```

**方式 4：认证保险库（凭据加密存储，按名称登录）**

```bash
echo "$PASSWORD" | agent-browser auth save myapp --url https://app.example.com/login --username user --password-stdin
agent-browser auth login myapp
```

`auth login` 会等待登录表单选择器出现后再进行填写/点击，在延迟加载的 SPA 登录页面上更可靠。

**方式 5：状态文件（手动保存/加载）**

```bash
# 登录后：
agent-browser state save ./auth.json
# 在未来的会话中：
agent-browser state load ./auth.json
agent-browser open https://app.example.com/dashboard
```

参见 [references/authentication.md](references/authentication.md) 了解 OAuth、双因素认证、基于 Cookie 的认证和令牌刷新模式。

## 基本命令

```bash
# 导航
agent-browser open <url>              # 导航（别名: goto, navigate）
agent-browser close                   # 关闭浏览器
agent-browser close --all             # 关闭所有活动会话

# 快照
agent-browser snapshot -i             # 带引用的交互元素（推荐）
agent-browser snapshot -s "#selector" # 限定到 CSS 选择器范围

# 交互（使用快照中的 @引用）
agent-browser click @e1               # 点击元素
agent-browser click @e1 --new-tab     # 点击并在新标签页打开
agent-browser fill @e2 "text"         # 清除并输入文本
agent-browser type @e2 "text"         # 输入文本（不清除）
agent-browser select @e1 "option"     # 选择下拉选项
agent-browser check @e1               # 勾选复选框
agent-browser press Enter             # 按键
agent-browser keyboard type "text"    # 在当前焦点处输入（无需选择器）
agent-browser keyboard inserttext "text"  # 插入文本（不触发按键事件）
agent-browser scroll down 500         # 滚动页面
agent-browser scroll down 500 --selector "div.content"  # 在特定容器内滚动

# 获取信息
agent-browser get text @e1            # 获取元素文本
agent-browser get url                 # 获取当前 URL
agent-browser get title               # 获取页面标题
agent-browser get cdp-url             # 获取 CDP WebSocket URL

# 等待
agent-browser wait @e1                # 等待元素出现
agent-browser wait --load networkidle # 等待网络空闲
agent-browser wait --url "**/page"    # 等待 URL 模式匹配
agent-browser wait 2000               # 等待毫秒数
agent-browser wait --text "Welcome"    # 等待文本出现（子字符串匹配）
agent-browser wait --fn "!document.body.innerText.includes('Loading...')"  # 等待文本消失
agent-browser wait "#spinner" --state hidden  # 等待元素消失

# 下载
agent-browser download @e1 ./file.pdf          # 点击元素触发下载
agent-browser wait --download ./output.zip     # 等待任何下载完成
agent-browser --download-path ./downloads open <url>  # 设置默认下载目录

# 网络
agent-browser network requests                 # 检查已跟踪的请求
agent-browser network requests --type xhr,fetch  # 按资源类型过滤
agent-browser network requests --method POST   # 按 HTTP 方法过滤
agent-browser network requests --status 2xx    # 按状态过滤（200, 2xx, 400-499）
agent-browser network request <requestId>      # 查看完整请求/响应详情
agent-browser network route "**/api/*" --abort  # 阻止匹配的请求
agent-browser network har start                # 开始 HAR 录制
agent-browser network har stop ./capture.har   # 停止并保存 HAR 文件

# 视口和设备模拟
agent-browser set viewport 1920 1080          # 设置视口大小（默认: 1280x720）
agent-browser set viewport 1920 1080 2        # 2 倍视网膜（相同 CSS 尺寸，更高分辨率截图）
agent-browser set device "iPhone 14"          # 模拟设备（视口 + 用户代理）

# 捕获
agent-browser screenshot              # 截图到临时目录
agent-browser screenshot --full       # 全页面截图
agent-browser screenshot --annotate   # 带编号元素标签的注释截图
agent-browser screenshot --screenshot-dir ./shots  # 保存到自定义目录
agent-browser screenshot --screenshot-format jpeg --screenshot-quality 80
agent-browser pdf output.pdf          # 保存为 PDF

# 实时预览/流
agent-browser stream enable           # 在自动选择的端口上启动运行时 WebSocket 流
agent-browser stream enable --port 9223  # 绑定到特定本地主机端口
agent-browser stream status           # 检查启用状态、端口、连接和屏幕广播
agent-browser stream disable          # 停止运行时流并删除 .stream 元数据文件

# 剪贴板
agent-browser clipboard read                      # 从剪贴板读取文本
agent-browser clipboard write "Hello, World!"     # 写入文本到剪贴板
agent-browser clipboard copy                      # 复制当前选中内容
agent-browser clipboard paste                     # 从剪贴板粘贴

# 对话框（alert, confirm, prompt, beforeunload）
# 默认情况下，alert 和 beforeunload 对话框会被自动接受，不会阻塞代理。
# confirm 和 prompt 对话框仍需显式处理。
# 使用 --no-auto-dialog（或 AGENT_BROWSER_NO_AUTO_DIALOG=1）禁用自动处理。
agent-browser dialog accept              # 接受对话框
agent-browser dialog accept "my input"   # 接受 prompt 对话框并输入文本
agent-browser dialog dismiss             # 关闭/取消对话框
agent-browser dialog status              # 检查当前是否有打开的对话框

# 差异（比较页面状态）
agent-browser diff snapshot                          # 比较当前与上次快照
agent-browser diff snapshot --baseline before.txt    # 比较当前与已保存文件
agent-browser diff screenshot --baseline before.png  # 可视化像素差异
agent-browser diff url <url1> <url2>                 # 比较两个页面
agent-browser diff url <url1> <url2> --wait-until networkidle  # 自定义等待策略
agent-browser diff url <url1> <url2> --selector "#main"  # 限定到元素范围
```

## 流

每个会话自动在操作系统分配的端口上启动 WebSocket 流服务器。使用 `agent-browser stream status` 查看绑定的端口和连接状态。使用 `stream disable` 关闭流，使用 `stream enable --port <port>` 在特定端口重新启用。

## 批量执行

通过将字符串数组的 JSON 数组管道传输到 `batch`，在单次调用中执行多个命令。这避免了在运行多步骤工作流时每个命令的进程启动开销。

```bash
echo '[
  ["open", "https://example.com"],
  ["snapshot", "-i"],
  ["click", "@e1"],
  ["screenshot", "result.png"]
]' | agent-browser batch --json

# 遇到第一个错误时停止
agent-browser batch --bail < commands.json
```

当您有一系列不依赖中间输出的已知命令序列时使用 `batch`。当需要在步骤之间解析输出时（例如，通过快照发现引用，然后交互），使用单独的命令或 `&&` 链式调用。

## 常见模式

### 表单提交

```bash
agent-browser open https://example.com/signup
agent-browser snapshot -i
agent-browser fill @e1 "Jane Doe"
agent-browser fill @e2 "jane@example.com"
agent-browser select @e3 "California"
agent-browser check @e4
agent-browser click @e5
agent-browser wait --load networkidle
```

### 使用认证保险库进行认证（推荐）

```bash
# 保存凭据一次（使用 AGENT_BROWSER_ENCRYPTION_KEY 加密）
# 推荐：通过 stdin 管道传输密码以避免 shell 历史暴露
echo "pass" | agent-browser auth save github --url https://github.com/login --username user --password-stdin

# 使用保存的配置文件登录（LLM 永远看不到密码）
agent-browser auth login github

# 列出/查看/删除配置文件
agent-browser auth list
agent-browser auth show github
agent-browser auth delete github
```

`auth login` 会在交互前等待用户名/密码/提交选择器出现，超时时间与默认操作超时绑定。

### 使用状态持久化进行认证

```bash
# 登录一次并保存状态
agent-browser open https://app.example.com/login
agent-browser snapshot -i
agent-browser fill @e1 "$USERNAME"
agent-browser fill @e2 "$PASSWORD"
agent-browser click @e3
agent-browser wait --url "**/dashboard"
agent-browser state save auth.json

# 在未来的会话中复用
agent-browser state load auth.json
agent-browser open https://app.example.com/dashboard
```

### 会话持久化

```bash
# 在浏览器重启时自动保存/恢复 cookies 和 localStorage
agent-browser --session-name myapp open https://app.example.com/login
# ... 登录流程 ...
agent-browser close  # 状态自动保存到 ~/.agent-browser/sessions/

# 下次使用时，状态自动加载
agent-browser --session-name myapp open https://app.example.com/dashboard

# 静态加密状态
export AGENT_BROWSER_ENCRYPTION_KEY=$(openssl rand -hex 32)
agent-browser --session-name secure open https://app.example.com

# 管理已保存的状态
agent-browser state list
agent-browser state show myapp-default.json
agent-browser state clear myapp
agent-browser state clean --older-than 7
```

### 处理 Iframe

Iframe 内容会自动内联到快照中。iframe 内的引用携带帧上下文，因此您可以直接与它们交互。

```bash
agent-browser open https://example.com/checkout
agent-browser snapshot -i
# @e1 [heading] "Checkout"
# @e2 [Iframe] "payment-frame"
#   @e3 [input] "Card number"
#   @e4 [input] "Expiry"
#   @e5 [button] "Pay"

# 直接交互——无需切换帧
agent-browser fill @e3 "4111111111111111"
agent-browser fill @e4 "12/28"
agent-browser click @e5

# 将快照限定到单个 iframe：
agent-browser frame @e2
agent-browser snapshot -i         # 仅 iframe 内容
agent-browser frame main          # 返回主帧
```

### 数据提取

```bash
agent-browser open https://example.com/products
agent-browser snapshot -i
agent-browser get text @e5           # 获取特定元素文本
agent-browser get text body > page.txt  # 获取所有页面文本

# JSON 输出用于解析
agent-browser snapshot -i --json
agent-browser get text @e1 --json
```

### 并行会话

```bash
agent-browser --session site1 open https://site-a.com
agent-browser --session site2 open https://site-b.com

agent-browser --session site1 snapshot -i
agent-browser --session site2 snapshot -i

agent-browser session list
```

### 连接到现有 Chrome

```bash
# 自动发现已启用远程调试的运行中 Chrome
agent-browser --auto-connect open https://example.com
agent-browser --auto-connect snapshot

# 或使用明确的 CDP 端口
agent-browser --cdp 9222 snapshot
```

自动连接通过 `DevToolsActivePort`、常用调试端口（9222, 9229）发现 Chrome，如果基于 HTTP 的 CDP 发现失败，则回退到直接 WebSocket 连接。

### 配色方案（深色模式）

```bash
# 通过标志持久化深色模式（应用于所有页面和新标签页）
agent-browser --color-scheme dark open https://example.com

# 或通过环境变量
AGENT_BROWSER_COLOR_SCHEME=dark agent-browser open https://example.com

# 或在会话期间设置（对后续命令持久生效）
agent-browser set media dark
```

### 视口和响应式测试

```bash
# 设置自定义视口大小（默认为 1280x720）
agent-browser set viewport 1920 1080
agent-browser screenshot desktop.png

# 测试移动端宽度布局
agent-browser set viewport 375 812
agent-browser screenshot mobile.png

# 视网膜/高 DPI：相同的 CSS 布局，2 倍像素密度
# 截图保持逻辑视口大小，但内容以更高 DPI 渲染
agent-browser set viewport 1920 1080 2
agent-browser screenshot retina.png

# 设备模拟（一步设置视口 + 用户代理）
agent-browser set device "iPhone 14"
agent-browser screenshot device.png
```

`scale` 参数（第三个参数）设置 `window.devicePixelRatio` 而不改变 CSS 布局。在测试视网膜渲染或捕获更高分辨率截图时使用。

### 可视化浏览器（调试）

```bash
agent-browser --headed open https://example.com
agent-browser highlight @e1          # 高亮元素
agent-browser inspect                # 为活动页面打开 Chrome DevTools
agent-browser record start demo.webm # 录制会话
agent-browser profiler start         # 开始 Chrome DevTools 性能分析
agent-browser profiler stop trace.json # 停止并保存性能分析（路径可选）
```

使用 `AGENT_BROWSER_HEADED=1` 通过环境变量启用可视化模式。浏览器扩展在可视化和无头模式下均可工作。

### 本地文件（PDF、HTML）

```bash
# 使用 file:// URL 打开本地文件
agent-browser --allow-file-access open file:///path/to/document.pdf
agent-browser --allow-file-access open file:///path/to/page.html
agent-browser screenshot output.png
```

### iOS 模拟器（移动端 Safari）

```bash
# 列出可用的 iOS 模拟器
agent-browser device list

# 在特定设备上启动 Safari
agent-browser -p ios --device "iPhone 16 Pro" open https://example.com

# 与桌面端相同的工作流程——快照、交互、重新快照
agent-browser -p ios snapshot -i
agent-browser -p ios tap @e1          # 点击（click 的别名）
agent-browser -p ios fill @e2 "text"
agent-browser -p ios swipe up         # 移动端专用手势

# 截图
agent-browser -p ios screenshot mobile.png

# 关闭会话（关闭模拟器）
agent-browser -p ios close
```

**要求：** macOS + Xcode + Appium（`npm install -g appium && appium driver install xcuitest`）

**真实设备：** 如果预先配置好，可以与物理 iOS 设备配合使用。使用 `--device "<UDID>"`，其中 UDID 来自 `xcrun xctrace list devices`。

## 安全性

所有安全功能都是可选的。默认情况下，agent-browser 不对导航、操作或输出施加任何限制。

### 内容边界（推荐用于 AI 代理）

启用 `--content-boundaries` 以在标记中包装页面源输出，帮助 LLM 区分工具输出和不受信任的页面内容：

```bash
export AGENT_BROWSER_CONTENT_BOUNDARIES=1
agent-browser snapshot
# 输出：
# --- AGENT_BROWSER_PAGE_CONTENT nonce=<hex> origin=https://example.com ---
# [无障碍树]
# --- END_AGENT_BROWSER_PAGE_CONTENT nonce=<hex> ---
```

### 域名白名单

将导航限制到受信任的域名。通配符如 `*.example.com` 也会匹配裸域 `example.com`。对非允许域名的子资源请求、WebSocket 和 EventSource 连接也会被阻止。请包含目标页面依赖的 CDN 域名：

```bash
export AGENT_BROWSER_ALLOWED_DOMAINS="example.com,*.example.com"
agent-browser open https://example.com        # 允许
agent-browser open https://malicious.com       # 被阻止
```

### 操作策略

使用策略文件来控制破坏性操作：

```bash
export AGENT_BROWSER_ACTION_POLICY=./policy.json
```

示例 `policy.json`：

```json
{ "default": "deny", "allow": ["navigate", "snapshot", "click", "scroll", "wait", "get"] }
```

认证保险库操作（`auth login` 等）绕过操作策略，但域名白名单仍然适用。

### 输出限制

防止大页面导致上下文溢出：

```bash
export AGENT_BROWSER_MAX_OUTPUT=50000
```

## 差异比较（验证更改）

执行操作后使用 `diff snapshot` 验证是否达到预期效果。这将比较当前无障碍树与会话中上次快照的差异。

```bash
# 典型工作流程：快照 -> 操作 -> 差异比较
agent-browser snapshot -i          # 拍摄基线快照
agent-browser click @e2            # 执行操作
agent-browser diff snapshot        # 查看更改内容（自动与上次快照比较）
```

用于视觉回归测试或监控：

```bash
# 保存基线截图，之后比较
agent-browser screenshot baseline.png
# ... 时间推移或发生更改 ...
agent-browser diff screenshot --baseline baseline.png

# 比较预发布环境与生产环境
agent-browser diff url https://staging.example.com https://prod.example.com --screenshot
```

`diff snapshot` 输出使用 `+` 表示新增，`-` 表示删除，类似于 git diff。`diff screenshot` 生成差异图像，更改的像素以红色高亮显示，并显示不匹配百分比。

## 超时和慢页面

默认超时为 25 秒。可通过 `AGENT_BROWSER_DEFAULT_TIMEOUT` 环境变量覆盖（值以毫秒为单位）。对于慢速网站或大页面，使用显式等待而不是依赖默认超时：

```bash
# 等待网络活动稳定（最适合慢页面）
agent-browser wait --load networkidle

# 等待特定元素出现
agent-browser wait "#content"
agent-browser wait @e1

# 等待特定 URL 模式（在重定向后很有用）
agent-browser wait --url "**/dashboard"

# 等待 JavaScript 条件
agent-browser wait --fn "document.readyState === 'complete'"

# 作为最后手段等待固定时长（毫秒）
agent-browser wait 5000
```

处理持续缓慢的网站时，在 `open` 后使用 `wait --load networkidle` 确保页面完全加载后再拍摄快照。如果特定元素渲染缓慢，直接使用 `wait <选择器>` 或 `wait @引用` 等待它。

## JavaScript 对话框（alert / confirm / prompt）

当页面打开 JavaScript 对话框（`alert()`、`confirm()` 或 `prompt()`）时，它会阻止所有其他浏览器命令（快照、截图、点击等），直到对话框被关闭。如果命令开始意外超时，请检查是否有待处理的对话框：

```bash
# 检查是否有对话框阻塞
agent-browser dialog status

# 接受对话框（关闭 alert / 点击确定）
agent-browser dialog accept

# 接受 prompt 对话框并输入文本
agent-browser dialog accept "my input"

# 关闭对话框（点击取消）
agent-browser dialog dismiss
```

当有对话框待处理时，所有命令响应都包含一个 `warning` 字段，指示对话框类型和消息。在 `--json` 模式下，这表现为响应对象中的 `"warning"` 键。

## 会话管理和清理

并发运行多个代理或自动化时，始终使用命名会话以避免冲突：

```bash
# 每个代理获得自己的隔离会话
agent-browser --session agent1 open site-a.com
agent-browser --session agent2 open site-b.com

# 检查活动会话
agent-browser session list
```

完成后始终关闭浏览器会话以避免进程泄漏：

```bash
agent-browser close                    # 关闭默认会话
agent-browser --session agent1 close   # 关闭特定会话
agent-browser close --all              # 关闭所有活动会话
```

如果之前的会话未正确关闭，守护进程可能仍在运行。使用 `agent-browser close` 清理它，或使用 `agent-browser close --all` 立即关闭所有会话。

要在一段时间不活动后自动关闭守护进程（适用于临时/CI 环境）：

```bash
AGENT_BROWSER_IDLE_TIMEOUT_MS=60000 agent-browser open example.com
```

## 引用生命周期（重要）

引用（`@e1`、`@e2` 等）在页面更改时会失效。在以下情况后必须重新快照：

- 点击导航链接或按钮
- 表单提交
- 动态内容加载（下拉菜单、模态框）

```bash
agent-browser click @e5              # 导航到新页面
agent-browser snapshot -i            # 必须重新快照
agent-browser click @e1              # 使用新引用
```

## 注释截图（视觉模式）

使用 `--annotate` 拍摄带有编号标签覆盖在交互元素上的截图。每个标签 `[N]` 对应引用 `@eN`。这也会缓存引用，因此您可以立即与元素交互，无需单独的快照。

```bash
agent-browser screenshot --annotate
# 输出包括图像路径和图例：
#   [1] @e1 button "Submit"
#   [2] @e2 link "Home"
#   [3] @e3 textbox "Email"
agent-browser click @e2              # 使用注释截图中的引用进行点击
```

在以下情况使用注释截图：

- 页面有未标记的图标按钮或仅视觉元素
- 需要验证视觉布局或样式
- 存在 Canvas 或图表元素（对文本快照不可见）
- 需要对元素位置进行空间推理

## 语义定位器（引用的替代方案）

当引用不可用或不可靠时，使用语义定位器：

```bash
agent-browser find text "Sign In" click
agent-browser find label "Email" fill "user@test.com"
agent-browser find role button click --name "Submit"
agent-browser find placeholder "Search" type "query"
agent-browser find testid "submit-btn" click
```

## JavaScript 执行（eval）

使用 `eval` 在浏览器上下文中运行 JavaScript。**Shell 引号可能会破坏复杂表达式**——使用 `--stdin` 或 `-b` 避免问题。

```bash
# 简单表达式使用常规引号即可
agent-browser eval 'document.title'
agent-browser eval 'document.querySelectorAll("img").length'

# 复杂 JS：使用 --stdin 配合 heredoc（推荐）
agent-browser eval --stdin <<'EVALEOF'
JSON.stringify(
  Array.from(document.querySelectorAll("img"))
    .filter(i => !i.alt)
    .map(i => ({ src: i.src.split("/").pop(), width: i.width }))
)
EVALEOF

# 替代方案：base64 编码（避免所有 shell 转义问题）
agent-browser eval -b "$(echo -n 'Array.from(document.querySelectorAll("a")).map(a => a.href)' | base64)"
```

**为什么这很重要：** 当 shell 处理您的命令时，内部双引号、`!` 字符（历史扩展）、反引号和 `$()` 都可能在 JavaScript 到达 agent-browser 之前破坏它。`--stdin` 和 `-b` 标志完全绕过 shell 解释。

**经验法则：**

- 单行、无嵌套引号 -> 常规 `eval 'expression'` 使用单引号即可
- 嵌套引号、箭头函数、模板字面量或多行 -> 使用 `eval --stdin <<'EVALEOF'`
- 程序化/生成的脚本 -> 使用 `eval -b` 配合 base64

## 配置文件

在项目根目录创建 `agent-browser.json` 进行持久化设置：

```json
{
  "headed": true,
  "proxy": "http://localhost:8080",
  "profile": "./browser-data"
}
```

优先级（从低到高）：`~/.agent-browser/config.json` < `./agent-browser.json` < 环境变量 < CLI 标志。使用 `--config <路径>` 或 `AGENT_BROWSER_CONFIG` 环境变量指定自定义配置文件（如果缺失/无效则报错退出）。所有 CLI 选项映射为 camelCase 键（例如 `--executable-path` -> `"executablePath"`）。布尔标志接受 `true`/`false` 值（例如 `--headed false` 覆盖配置）。用户和项目配置中的扩展被合并，而非替换。

## 深入文档

| 参考文档                                                          | 使用场景                                         |
| -------------------------------------------------------------------- | --------------------------------------------------------- |
| [references/commands.md](references/commands.md)                     | 完整命令参考及所有选项                   |
| [references/snapshot-refs.md](references/snapshot-refs.md)           | 引用生命周期、失效规则、故障排除        |
| [references/session-management.md](references/session-management.md) | 并行会话、状态持久化、并发抓取 |
| [references/authentication.md](references/authentication.md)         | 登录流程、OAuth、双因素认证处理、状态复用             |
| [references/video-recording.md](references/video-recording.md)       | 调试和文档的录制工作流程       |
| [references/profiling.md](references/profiling.md)                   | Chrome DevTools 性能分析        |
| [references/proxy-support.md](references/proxy-support.md)           | 代理配置、地理测试、轮换代理        |

## 浏览器引擎选择

使用 `--engine` 选择本地浏览器引擎。默认为 `chrome`。

```bash
# 使用 Lightpanda（快速无头浏览器，需单独安装）
agent-browser --engine lightpanda open example.com

# 通过环境变量
export AGENT_BROWSER_ENGINE=lightpanda
agent-browser open example.com

# 使用自定义二进制路径
agent-browser --engine lightpanda --executable-path /path/to/lightpanda open example.com
```

支持的引擎：
- `chrome`（默认）——通过 CDP 使用 Chrome/Chromium
- `lightpanda`——通过 CDP 使用 Lightpanda 无头浏览器（比 Chrome 快 10 倍，内存占用少 10 倍）

Lightpanda 不支持 `--extension`、`--profile`、`--state` 或 `--allow-file-access`。从 https://lightpanda.io/docs/open-source/installation 安装 Lightpanda。

## 可观测性仪表盘

仪表盘是一个独立的后台服务器，显示所有会话的实时浏览器视口、命令活动和控制台输出。

```bash
# 安装仪表盘（一次性）
agent-browser dashboard install

# 启动仪表盘服务器（后台运行，端口 4848）
agent-browser dashboard start

# 所有会话自动在仪表盘中可见
agent-browser open example.com

# 停止仪表盘
agent-browser dashboard stop
```

仪表盘在端口 4848（可通过 `--port` 配置）上独立于浏览器会话运行。所有会话自动流式传输到仪表盘。也可以从仪表盘 UI 使用本地引擎或云提供商创建会话。

## 即用模板

| 模板                                                                 | 描述                         |
| ------------------------------------------------------------------------ | ----------------------------------- |
| [templates/form-automation.sh](templates/form-automation.sh)             | 带验证的表单填写        |
| [templates/authenticated-session.sh](templates/authenticated-session.sh) | 登录一次，复用状态             |
| [templates/capture-workflow.sh](templates/capture-workflow.sh)           | 带截图的内容提取 |

```bash
./templates/form-automation.sh https://example.com/form
./templates/authenticated-session.sh https://app.example.com/login
./templates/capture-workflow.sh https://example.com ./output
```