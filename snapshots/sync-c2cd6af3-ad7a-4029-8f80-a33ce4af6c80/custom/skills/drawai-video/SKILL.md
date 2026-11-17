---
name: 'drawai-video'
description: '用 DrawAI 画布做视频：定镜头卡与整片风格、写分镜表、生成工程、出样片与成片；含品牌素材收集、配音与踩点。'
whenToUse: '用户让你做视频/宣传片/产品片/MG 动画、给已有视频改镜头或配音、把画布上的流程做成片子，或问"样片怎么出""镜头卡怎么连""配乐怎么踩点"时。'
---

# DrawAI 画布做视频：从问到出片

这个技能只讲**怎么把一条片子做出来**。画布怎么读、怎么改（diagram_read / diagram_apply 的完整签名、
折点与端点、图层与顺序、traceImage 位图矢量化）在技能 drawai-canvas 里 —— 视频流程真正用到的那几个 op
下面列全了，不够用时再加载那个技能。

## 画布最小集（视频只用到这些 op）

    // 1) 先读回当前那张画布：用户说"这张"就别猜路径，不传 path 读一次
    diagram_read()                       // → { path, absolute, nodes, edges, ... }，path 拿去给 apply
    // 2) 一次写完：建节点、连卡、放分镜表；几何已经规划好就 layout:"none"
    diagram_apply({ path, layout:"none", ops:[
      {op:"addNode", label:"分镜表", shape:"text", x:40, y:40, w:520, h:360, as:"board"},
      {op:"addNode", label:"主片", shape:"video", x:600, y:40, w:480, h:270, as:"film"},
      {op:"addEdge", from:"shot-card", to:"film"}   // shot-card 是画布上已存在的镜头卡 id
    ]})

- **path 必填**：diagram_apply 永远要显式传 path；diagram_read 不传时返回当前聚焦的那张。
- **from / to 必须是已存在的节点 id**（写错在写盘前就报错）；as 起的别名只在同一次 ops 数组里有效。
- **建了镜头卡就必须连**：断连的卡既没进 assets、也不排期，motion_project create 会直接报错。
- 连线用圆角折线，折点自己规划（points 只放中间转折，不放源/目标连接点），不要启用自动避障。
- 挪位置用 move，改文字用 setLabel，换样式用 setStyle；改一组就传 ids:[...]。改接边用 setEdge，
  不要 remove + addEdge 重画 —— 那会丢掉端点约束和折点。

## 视频节点与工作流编排

画一条视频分十步，执行者就是你自己。**第 0 步（问人）、品牌素材、定卡并连卡、分镜表、样片，这五步不能跳**。

0. 先问人，再动手 — **没问过风格就开工是最贵的错**。用 ask_user_question 一次问齐这四件事（不要每步都打断）：
   - 风格与气质：暗底开发感 / 明亮克制 / 高饱和冲击 / 复古纸质 / 电影感黑金……有没有参考片、参考站；
   - 主题与受众：给谁看（开发者、客户、投资人），最想让人记住的一件事是什么；
   - 规格：时长、画幅（16:9 横屏 / 9:16 竖屏）、要不要旁白与字幕、要不要真声音乐；
   - 品牌资产从哪来：官方 logo/字标、图标、产品界面截图 —— 用户直接给文件，还是授权你去官方站取。
   能推断的先推断，写成默认项一起问（"我按 16:9 / 30s / 不要旁白来做，除非你说别的"）。用户回"你定"就自己定，然后明确说你定了什么。**风格没定之前不要 create 工程**；答案要落进 BRIEF.md 与分镜表。
   问完不要直接开工：先给 **2–3 个风格方向**（每个 6 行以内：一句话概念 / 参考质感 / 运镜语汇 / 适合什么内容 / 不适合什么），让他选一个。给"暗底开发感 / 明亮克制"这种空词没有用，要给看得见的画面。

1. 素材与品牌（先收集，再动笔）— 片子里会出现的一切"真东西"都先收齐，不许凭印象画：
   - **要整个站点就用 gather**：motion_assets {action:"gather", url, brand, pages, limit, dryRun} 扫首页 + 导航里挑出的内页（默认 4 页），把 favicon、og:image、主视觉、产品截图、内联字标一次收齐，跨页聚合**实测配色**（hex + 出现次数）与字体，再补三条品牌 logo 兜底（Simple Icons 矢量 / Clearbit 512 / 高清 favicon），并给每页一张 2 倍视口截图。先 dryRun:true 看清单，再正式收（默认 12 张）。品牌名与域名都写不准时，只给 brand 也能先拿到 logo。
   - **只关心一个页面就用 collect**：{action:"collect", url}，同样的扫描，只扫这一页（默认 8 张）。
   - **官方品牌资产**：web_search 找「<品牌> brand guidelines / press kit / logo svg」→ read_page 读规范（最小留白、禁止变形、标准色、黑白版规则）→ logo 具体直链用 {action:"fetch", url} 取原始矢量（别截图、别重画）。
   - **用户能给文件就用用户的**（{action:"capture", path}）；界面要分层或只截一块，用 {action:"page", url} 再 {action:"crop"}。
   - 找不到官方素材就直说，把"用纯文字占位"当成待确认提案问用户 —— 不要默默换成一个"看起来像"的字体（一眼假，最掉价）；
   - 截图里影响版面的水印、乱码，用 crop 裁掉或换素材。
   收完先把返回的 colors 与 fonts 对进 theme.ts：**页面实测值**比凭感觉调色更像；fontSources 里 googleFonts 为 true 的可以照 cssUrl 去取真字体，其余退回系统字（在 BRIEF.md 里写明"字体非官方"）。素材不止图片：图标、纹理、产品界面、波形、色板都算，缺哪类就补哪一个 url。
   规矩：不改比例、不改描边、不重上色（官方允许的黑白版除外）；字标的大小写、字重、字距按官方来。

2. 定镜头表（设计，不写代码）— 先用 shotcraft_search 按关键词/分类找准确的卡名与样式；**对选中的卡再用 cardId+styleKey 搜一次**，把它的配方（recipe：参数表、动效核心、已知坑）和 demo 源码读回来。跳过这一步，卡就只是个标签。然后用 diagram_apply addNode {shape:"shotcard",cardId,styleKey,x,y,as:"shot"} 建卡、addEdge 从卡连到视频节点，连线顺序就是镜头顺序。**建了卡就必须连**：没连的卡不会进 assets、不参与排期，create 会直接报错——断连不是"只作参考"，是三样一起丢。
   如果这条片子要的是**整片视觉语言**（扁平矢量、线条、等轴、拼贴、包豪斯、像素、HUD…），先用 mg_style {action:"list"} 挑一种，{action:"read", slug} 读它的施工包（判据/色板/节奏/移植 brief），再 diagram_apply addNode {shape:"mgstyle", slug} 连到视频节点。两者分工：风格管整片（色板、字体、节奏、动效语法），镜头卡管单镜运动方式，可以同时用；一条片子只用一种整片风格。
   **引擎是硬约束**：read 返回里的 engines 就是上游真用的引擎（three / webgl2 / webgl / canvas2d / dom），engineFiles 是证据文件。上游用 three.js 或 WebGL2 的，移植必须真的跑同一套引擎——three 或 @remotion/three 在 canvas 上渲染，或原生 WebGL2 + GLSL；**不许用 CSS 3D、div 层叠、Canvas2D 渐变模糊「简化」替代**。STYLE.md 的「引擎契约」写了上游证据与逐引擎做法；motion_render 启动前会扫工程源码，重引擎缺失直接拒绝渲染。
   **动效也是硬约束**：read 返回里的 motion 是从上游源码抽出来的动效画像（转场是 wipe 还是切、有没有 impact/冲击环、相机漂移、motion blur 几采样、颗粒多粗、有哪些二次运动）。**写场景前先 mg_style {action:"frames", slug} 取 8 张参考帧，用 read_image 逐张看**——brief 只写「是什么」，帧才写「怎么动」。STYLE.md 的「效果配方」是必做项：只把形状做对、转场用淡入淡出、落地不压不抖、颗粒几百个点，出来的就是「形似而动不像」，这是最常见也最贵的返工。
3. 写分镜表（画布上是**一张表**，不是一镜一个节点）— motion_spec {action:"scaffold", nodeId, storyboard:[{intent 这一镜要做什么, onScreen 屏幕上出现什么, copy 配什么字, card 镜头卡}], style, narration} 返回把【分镜表】/【风格】/【旁白】摆到视频附近空位的 diagram_apply ops，ops 原样交给 diagram_apply（layout:"none"）。表是 Markdown 六列（镜头 | 帧区间 | 镜头卡 | 这一镜要做什么 | 屏幕上出现什么 | 文案 / 声音），**每一镜写成表里的一行**；帧区间写 `0–35f` 或 `1.5s`，各镜之和必须等于视频节点时长。
   **绝对不要一镜建一个文本节点**——那样 motion_project 读不到分镜，画布上也不再是一张表（motion_spec {action:"validate"} 会把这件事故直接报出来）。**没有分镜表不要开写场景**；功能点对不上镜头就先改表。
4. 建工程 — motion_project {action:"create", nodeId, spec:{width,height,fps,duration}, prompt}。没显式给 storyboard 时它**直接读画布上的【分镜表】**（所以上一步那张表就是真相源），按镜头卡排 timeline.json，段数=卡数；没挂卡时返回里会明确写「均分，未挂镜头卡」并给 warning。**想控制总时长不用断连**：显式给 spec.duration 时，卡片按相对时长等比缩放到总长，不会被卡的总和改短。
5. 样片（pilot）— 除非用户明确要求跳过，先做可在画布播放的样片，便于及时检查：
   - 先只写 scene-0，motion_preview {nodeId} 看一帧定调，把纸色、字重、留白、动效气质定住；不对就改 scene-0 重看，别写完四段再回头返工；
   - 挑能代表整片风格的 3–6 秒：对主视频节点调用 motion_render {action:"start", nodeId:"主视频节点", range:[起,止]}，记下返回的 pilot jobId；需要时用同一主视频节点和 jobId 调 status，确认 done。文件 .out/pilot.mp4 出现或静帧预览成功都不代表已展示给用户；
   - **样片渲染完成后必须主动回填**：在同一画布、同一页用 diagram_apply addNode {shape:"video", label:"样片", x, y, w:480, h:270} 新建节点并记住其 ID（已有合适样片节点时复用）；再调用 motion_render {action:"apply", nodeId:"样片节点 ID", jobId:"pilot jobId", source:"pilot"}。核对返回 kind=pilot、nodeId 和画布节点，向用户指出样片位置；失败则说明错误和文件路径。不要把样片写进主视频节点，主节点留给整片；
   - 样片要有真实排版、真实字重、真实配色、至少一处动效，**不是静态海报**；发给用户的是**渲染出来的 mp4**（回填到节点），不是静帧拼图——拼图会把 1920 缩到 650 宽，细线全糊，双方都会误判；
   - 用户要求先看样片再继续时，展示后等待反馈；用户已要求直接完成时可继续制作，不额外设置确认关卡。用户明确要求跳过样片时按其要求执行。
6. 写画面 — 用 read/edit 直接改 scene-<n>.tsx，对照读回来的 demo 源码复刻动作顺序、缓动、错峰。一场景一文件，单文件不超过 120 行；共享原语放 components/，上游组件可直接 import（别名 @shotcraft/lib/DigitRoll、@shotcraft/demos/<路径>）。场景里写相对本段的帧号，时间轴只改 timeline.json 的 startFrame/durationFrames。
7. 看图与补素材 — 改完必看，不要凭想象交付：motion_preview {nodeId, frames:[...]} 同一轮返回拼图，第一次要打包（10–30 秒），之后复用缓存只要几秒。**每镜写完做一次"同段对照"**：把 mg_style frames 的参考帧与 motion_preview 的成片帧并排看，按 STYLE.md「效果配方」逐项打勾（转场/冲击/相机/运动模糊/颗粒/二次运动），对不上就改，别等整片。缺真图时用 motion_assets：action:"capture" 把原始字节搬进 assets/（同时给尺寸与色板），action:"crop" 取局部，**action:"page" 用无头 Chrome 把一段 HTML 模板或网址截成 2 倍图**（写产品界面镜头用它，别自己先截图）。场景里 staticFile("文件名") 取。画布快照会被缩放重绘，精度要求高就走这一步。
8. 配声音与踩点 — 音频和画面同一条时间轴（timeline.json 的 audio[]），渲染时由 entry 统一挂 <Audio>，不是成片后再混。motion_audio {action:"add", nodeId, file 或 audioNodeId, role:"bgm"|"voice"|"sfx", at:秒, gainDb, fadeIn, fadeOut, duck} 放置音轨（素材没进工程会自动捕获）；role:"bgm" 默认 -10dB 且旁白自动避让。音效优先检查用户素材和本地库；不合适时再用 motion_sfx：search 按声音需求搜索，get 下载并保存音频索引，返回里的 suggestedGainDb 直接当 gainDb 用。踩点用 motion_audio {action:"analyze"} 量 BPM/网格残差/重音三分类/音源峰值滞后，bpm 为 null 就别硬对；action:"apply" 把切点吸到拍子上，带 cueId 可把某条音轨钉到某一帧（自动扣音源滞后与 1.29 帧输出偏移）。
9. 出片 — motion_render {action:"start"} 立刻返回 jobId，绝不等待；用 action:"status" 查进度。只渲几帧用 frames；range 渲出来的是**样片（pilot）**，不要拿它当整片。出片后 action:"apply" 把 film.mp4 回填成节点上的 clip pack（画布上直接能播）。
   **成片必须自己写回**：motion_qa 一过就立刻 motion_render {action:"apply", nodeId:"主视频节点 id"} 回填到主视频节点——apply 找的是最近一次**整片** film 任务，不会拿样片顶替；不要等用户开口要，他看不到工作区里的 mp4，只看得见画布上的节点。回填后海报取片长 35% 处（深色片子 0.4s 是黑帧，缩略图会像空节点）。

**最常翻车的六件事**：① 没问风格就开工，整片气质跑偏，返工最贵；② 拿系统字体假装品牌字标（用户一眼看出不是官方 logo）；③ 只有一张静帧就说"定调好了"，没出样片就让人看整片；④ 镜头卡建了却没连线——参考、顺序、段长一起丢，画布上看着"有卡"，工程里却没排期；⑤ 重引擎风格（three / WebGL2）用 CSS 或 Canvas2D 简化替代——静帧能骗、一动就露，render 会直接拦；⑥ 只做"形"不做"动"——形变过圆、转场靠淡入淡出、落地不压不抖、颗粒像噪点，没看参考帧就写场景，成品和上游差一个档。

素材工作流：用户提供的素材和品牌资源优先；需要纸纹、颗粒、胶带、敲击或乐器单音时，先用 motion_assets {action:"list",nodeId} 看工程素材，再用 {action:"library",nodeId,query:"纸张"} 或 query:"钢琴" / "颗粒" 按需查插件素材库；选中后调用 {action:"use",nodeId,assetId:"返回条目的 id"} 导入，只能用返回的 file。图片在场景里用 staticFile(file)，声音再用 motion_audio {action:"add",nodeId,file,role:"sfx",at:秒} 上时间轴，仅导入不会自动出现在画面或声音里。VCSL 乐器采样是短单音，不是完整背景音乐；检查电平并在样片中试听。现有素材不合适时再找其他来源。素材来源和许可见返回信息，不要将未逐项列明的许可描述为已确认。
插件素材随安装包提供，路径由 motion_assets 管理；上游路径只是素材 id，不是工作区中的磁盘路径。不要猜测插件或上游目录、从其他工程复制同名文件，也不要手写 assets/manifest.json。使用 library 返回的 id 调 use，由工具校验原文件并登记哈希和许可；导入失败时报告具体错误，排查插件安装或环境，不要静默绕过。
以上素材工程配合 motion_project / motion_render；motion_task 使用单独的画布引用素材快照，不会自动读取此工程的 assets。要给 motion_task 使用，需将对应素材导入画布节点并连接视频节点，或使用 motion_project / motion_render 完成制作。

音频素材：audio_asset search/get/import 添加音效、音乐或旁白节点；motion_audio add 也接受 file（本地 mp3/wav/m4a）或画布上的音频节点。生成音乐、TTS 与文本生成音效服务尚未接入。
镜头卡是设计参考，**不进成片画面**；但必须连到视频节点——连线决定它进不进 assets、排第几镜、这一镜多长。想换顺序改连线（或同时改 timeline.json），不要改场景文件里的绝对帧号。
**引擎保真**：选了带引擎的风格，顺序是 读 engines/engineFiles → 用 mg_style action:"source" 读上游那套引擎的实现文件 → 在 scene-*.tsx 里真跑这套引擎（three/@remotion/three、原生 WebGL2+GLSL、或该风格本来就用的 Canvas2D/DOM）→ motion_preview 看帧 → 再出样片。brief 是移植说明，不是换实现的许可；确实要换，先在 STYLE.md 写「偏离：引擎 X 改为 Y，因为 Z」，render 会带 warning 通过。用 CSS/div/静态图冒充实 3D 或 shader，等于把这个风格的判据扔掉。
**画布布局与连线**：先用 diagram_read 查看已有节点、画布尺寸和连线，保留用户已有的位置；按阅读方向分组，同类节点对齐、留出节点和文字间距，并为连线预留清晰通道。视频规格、视频/样片与镜头卡可分区，空间不足时选择另一侧或下方的空位。工作流用圆角折线，**不要使用自动避障**：AI 先确定节点实际 x/y/w/h，再选择合适的 exit/entry，根据画布坐标亲自规划水平/垂直通道和中间折点，以 addEdge {line:"rounded", points:[{x,y},...]} 写入；折点不包含端点。节点或参考的实际位置尚未确定时，先完成布局、diagram_read 读回坐标，再用 setEdge {id,points:[...]} 写入折点。必要时扩大间距，主动检查折线不穿节点或文字、不产生多余折返、交叉或重叠；对齐而不需要转弯时 points 可为空。使用 layout:"none" 保留已规划几何；用户要求自动布局时可先布局再重新规划连线。调整已有自动线时先 setStyle {avoid:false}，再 setEdge 指定折点；保留其他用户明确自定义的线型和未要求修改的手工路径。
**折点与端点分开**：exit/entry 决定节点连接侧，插件会补端点及引出段；points 只填确实需要的中间转折，绝不放源/目标连接点或重复折点。例：A=(60,60,240,140) 右侧中点为 (300,130)，B=(400,40,480,270) 左侧中点为 (400,175)；从 A 右侧到 B 左侧可用 exit:"e",entry:"w",points:[{x:340,y:175}]，不要再加 {x:400,y:175}。若起终点沿进出方向对齐且通道畅通，使用 points:[]。连线后先读回坐标和折点，具备图像通道时用 diagram_read {render:true} 查看实际画布；出现折返、穿线或覆盖文字时用 setEdge 修正，再复查。预览不可用时如实说明未做视觉校验。
参考图：diagram_apply 连接图片节点后会把画布快照冻结进工程 assets/，场景里通过 props.assets 取用。视频参数（宽高/fps/duration）在 motion_project create 时写入节点；用户指定时长必须显式传 duration（支持 9.3 等小数秒），不能只写在 prompt 里。
旧任务记录用 motion_task list/status/cancel/apply/focus/remove 管理，不要再指望它 start。多页或多个视频时明确指定 page/nodeId。
不要把这条流程想成"提交一个任务等结果"：你是在一个普通工程里写代码、看图、渲染，每一步都由本次对话直接完成。
