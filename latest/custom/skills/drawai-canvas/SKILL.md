---
name: 'drawai-canvas'
description: '读写 DrawAI 画布（.drawio）：diagram_read 看现状、diagram_apply 结构化改图；支持图像复刻、视频分段复刻及 SVG 帧序列导出。'
whenToUse: '用户让你画流程图/架构图/泳道图、复刻图片或视频、将视频转为 SVG 帧动画，或让你读/改某张 .drawio 画布，或问"这张图里有什么""画布上怎么操作"时。'
---

# DrawAI 画布：怎么读、怎么改

## 一句话
工作区里任何一个 .drawio 文件**就是**一张 DrawAI 画布 —— 它是 drawio 自己的 mxfile 格式。
用户在 DSH 右栏画布面板里看到的、以及用 drawio 桌面版打开的，是同一份文件；
没有"导入/导出"这一步，你直接读写这个文件就是读写那张画布。

视频节点、镜头卡、分镜表也都是这张画布上的对象，但**做视频**（选卡、写分镜表、出样片与成片）
是另一套流程：加载技能 drawai-video。这里只讲怎么读写它们。

## 两个工具怎么配合
1. **diagram_read(path?, layer?, ids?)** —— 先读。返回节点（id/label/shape/style/**x·y·w·h 坐标尺寸**）、
   边（id/from/to/label/style/dash/arrow/折点/悬空端的自由点/标签位置 labelX·labelY）、
   revision（**文件内容指纹**，不是版本号）、page（**纸张尺寸**：文件里写了才有）、
   layers（**图层表**：顺序 = 叠放顺序，含 name/visible/locked）、每个单元的 layer（它在哪一层）、
   notes（画布表示不了但会原样保留的东西：多页、分组、图片、HTML 标签）、
   **selection（用户此刻选中的单元 id）**、highlight（AI 自己请求的高亮，见下）、
   canRevert / revertSteps（AI 的改动还能退回几步，见下）、export（导出状态）。
   fontColor（设过才有）是**字色**的派生名：命中调色板给名字（red/grey…），否则给十六进制。
   visible:false 的层在画布上不画，但里面的单元**照旧读得到** —— 隐藏是显示状态，不是删除；
   想让人看见就提醒用户去「图层」菜单打开。
   **大图先过滤**：layer（层 id 或层名）只看一层、ids 只看几个单元 —— 过滤时返回体会带
   filtered 与 totalNodes/totalEdges（整张有多少），层表照旧整份回。别一上来就把几百个单元读完。
2. **diagram_apply(path, ops, layout?, expectRevision?)** —— 再改。给一组结构化编辑，宿主**无损写回**：只改我们拥有的单元，
   文件其余部分（别的页、未知单元、自定义属性）逐字节保留。打开后原样保存 = 文件一个字节都不变。

写入必须显式传 path。用户说"这张图"时，先用 diagram_read 读取当前聚焦画布，再把返回的 path 传给 diagram_apply；无法确认目标时先打开画布。改完可以再 read 一次自查。

**当前是哪一张，有两个来源**：用户在画布面板里切换标签页时，宿主会把"当前画布 = <路径>"
**注入到会话里**（你会看到一条 plugin 来源的上下文）；不放心时也不带 path 调一次
diagram_read —— 返回里的 path 就是它（这两条是一致的，同一个记录）。

**"这几个"是哪几个**：用户在画布上选中的单元会随 diagram_read 一起回来（selection）。
用户说"把这几个挪一下/换个颜色"时就用那些 id —— 别靠坐标猜。选中的是连线也一样能用
（setStyle/move/remove 都收边 id）。没人选时它是空数组。

**读到的 revision 别扔**：下次改这张图时把它当 expectRevision 传进来。对不上（用户在 drawio 里
改过、或另一个会话动过）调用会**直接失败、一个字节都不写** —— 重新 read 一次拿新指纹再改。
不传就是不检查（并发编辑时可能覆盖别人的改动，所以"读过再改"的流程里最好都传）。

手动布线：先读取或确定节点位置，按画布实际 x/y/w/h 为每条工作流边设计避开节点和文字的水平/垂直通道；addEdge / setEdge 用 points:[{x,y},...] 按源端到目标端顺序指定中间折点（不含端点），绝对画布坐标、保留小数、不吸附网格，最多1000个。显式 points 切入手动模式；不要同时给 avoid:true。points:[] 清空并保持手动，省略 points 保留原折点。仅用户明确要求自动避障时使用 setStyle{avoid:true,clearPoints:true}；仅用户明确要求非正交逐段连线时使用 line:"straight"。先确定节点位置，再用 layout:"none" 写入并复查。

## ops 速查
    {op:"addNode", label?, as?, shape?, svg?, style?, keys?, data?, w?, h?, x?, y?, fontColor?}
    {op:"addEdge", from?, to?, fromPoint?, toPoint?, as?, label?, points?, style?, dash?, arrow?, color?, exit?, entry?, jettySize?, edgeStyle?, line?, avoid?, fontSize?, fontColor?, keys?}
    {op:"setLabel", id|ids, label}
    {op:"setLabelPos", id, x?, y?, dx?, dy?, center?}
    {op:"setStyle", id|ids, shape?, svg?, style?, keys?, w?, h?, dash?, arrow?, color?, exit?, entry?, jettySize?, edgeStyle?, line?, avoid?, fontSize?, fontColor?, clearPoints?}
    {op:"setEdge", id, from?, to?, fromPoint?, toPoint?, points?}
    {op:"order", id|ids, to:"front"|"back"|"up"|"down"}
    {op:"duplicate", id|ids, dx?, dy?, withEdges?, layer?, as?}
    {op:"addLayer", name?, id?, visible?, as?}
    {op:"setLayer", id|ids, layer}
    {op:"setLayerProps", layer, name?, visible?, locked?}
    {op:"export", format?:"svg"|"png", name?}
    {op:"traceImage", image, removeBg, preset?, w?, h?, x?, y?, colors?, colorsDark?, colorsFill?, grid?, gridH?, scale?, epsilon?, minArea?, minRegionArea?, protectLuma?, majorityContrast?, bgColor?, bgSolidTol?, bgSolidBorderRatio?, cropX?, cropY?, cropW?, cropH?, as?, layer?}
    {op:"move", id|ids, dx?, dy?, x?, y?}
    {op:"remove", id|ids}
    {op:"highlight", ids:["n1","n2"]}

- **改接一条边用 setEdge，别 remove + addEdge 重画**：setEdge 只换 source/target（from/to 给节点 id，
  或 fromPoint/toPoint 给绝对点 = 脱开节点变成自由端），边的 **id、标签、标签位置、折点、
  端点约束（exitX/entryX）全都留着** —— 重画会把这些丢掉，还会让别的引用它的地方失效。
- **边上的文字压住线**：diagram_read 回 labelX/labelY（拖过才有；x = 沿边比例 -1..1，0 = 弧长中点、
  y = 垂直偏移 px）。{op:"setLabelPos", id, center:true} 放回中点（= 画布右键「标签居中」）；
  {op:"setLabelPos", id, dy:12} 往上/下挪 12px（dx 是比例，一般别用）。
- **顺序（谁压谁）用 order**：to 是 front（到最上面）/back/up/down，与画布右键的顺序菜单同一套。
  它改的是**文件里单元的先后**（drawio 打开也是这个层叠）。注意"所有边都画在所有节点下面"
  是画布目前的层叠限制，所以"让边压住节点"要放到更上面的**图层**里，不是 order 能解决的。
- **再来一份用 duplicate**：新 id + 整体平移（默认两格 20px，与画布 Ctrl+V 一样），
  折点、端点约束、标签位置、自定义数据都带过去。默认会把"两端都在被复制节点之间"的边一起复制
  （withEdges:false 只要节点）；只列一条接在节点上的边会被拒 —— 把两端节点一起写进 ids。
- **图层**：addLayer 新建一层（加在最上面，返回它的 id）、addNode/addEdge/duplicate 可以带 layer
  （层 id 或层名）、setLayer 把已有单元移到别的层、setLayerProps 改层的名字/显示/锁定。
  隐藏一层是**文档状态**（写进文件，drawio 打开也是隐藏的），不是删除 —— 隐藏层里的单元照旧读得到。
- **导出**：{op:"export", format:"svg"} 会让画布渲染一张 SVG 落在 .drawio 旁边（同名的 .svg，
  可以用 name 指定基名）；format:"png" 则是**浏览器下载**一张 PNG（用户下载目录里）。
  渲染在画布那一半做，所以这一步是"请求"：返回值/下一次 read 里的 export 状态
  pending → done（带文件路径）/ downloaded / failed。看到 pending 说明画布还没刷新 ——
  别反复请求，等下一次 read 再看（或提醒用户打开那张画布）。
- **把一张位图"描"成矢量图用 traceImage**：{op:"traceImage", image:"ref.png", removeBg:false, x:40, y:40}。
  它**真的读那张图的像素**，建成一个 svg 内容节点。
  **不要用"手写一段近似的 svg"冒充复刻** —— 形状词汇表表达不了的照片/插画就直接用这个 op。
  · 默认 **w/h = 原图像素尺寸**、viewBox = 原图像素坐标系（1:1 铺满）。
  · **先选 preset，再按需覆盖单字段**（合并：默认 ← preset ← 显式字段）：
    - preset:"lineart" 插画/线稿（多暗色配额、严保细线）
    - preset:"photo" 照片/渐变
    - preset:"simple" 图标/简单图（少色+降采样，省字符）
    - 不写 = "default"
  · **removeBg 必填**（无 auto）：true = 抠背景（先识纯色底，否则近白泛洪）；false = 整图保留。
    白底/纯色底图标、透明 PNG 通常 true；风景/照片/带场景背景通常 false。回执看 bgRemoved / bgDetect。
  · 调参闭环：看返回值里的 **trace[]**（结构化指标 + hint），不满意就 **remove 旧节点再 traceImage**（改 preset/字段）。
    常见：字符紧 → scale:2 / 加大 epsilon / preset:"simple"；线糊断 → preset:"lineart" 或提高 protectLuma/colorsDark。
  · 单次上限：SVG 标记 250000 字符（整图的精细度主要由它决定：预算紧时管线会抬 epsilon、丢小环，
    把细节压掉——想要更细也可以给 maxBytes 顶到上限，或干脆用下面的裁剪只描局部）；
    输入 PNG（零依赖）与 JPEG/GIF/WebP（需 sharp）。
  · **五官/小物件糊成一团时，别只调参 —— 用 cropX/cropY/cropW/cropH 裁出那一块单独描**：
    工作分辨率上限是 grid（默认 499px 宽），整张立绘塞进去时"脸"只占三四十像素，眼睛只有 3~4px，
    量化后五官必然消失（实测把 colors 拉满 24、protectLuma 提到 235 也救不回来）。
    裁出头部再描，同一块就占满工作图宽度 —— 脸能有一两百像素，五官才留得住。
    裁剪块的 SVG 使用**裁剪局部系**，节点框负责缩放和放置。
    默认 w/h 为裁剪块原始尺寸，x/y 为裁剪原点；显式 w/h 是裁剪块目标尺寸。
    原图若已缩放，x/y 与 w/h 都需按同一比例换算，可与整图那块叠着放（顺序在后 = 画在上面）。

- **改一组用 ids**：{op:"setStyle", ids:["n1","n2","n5"], style:"green"} —— setStyle / setLabel /
  move / remove 都收 ids，不要为了"把这几个改成绿色"发 12 个 op。批量 move 只收 dx/dy
  （给一个绝对 x/y 等于把它们叠成一摞，宿主会直接拒绝）。
- **addNode 的 label 可省或给 ""**：装饰面板 / 底框 / 纯色块不需要字时直接省掉；空 label 时请显式给 w/h
  （否则按最短标签估宽 130）。有字的节点照常写 label。
- **引用刚建的那个用 as**：{op:"addNode", label:"开始", as:"start"} 之后，同一个 ops 数组里
  from/to/id 可以直接写 "start"。id 省略时是自动分配的（n 几/e 几），猜编号在非空画布上必错。
- **返回值里看 created/changed/removed**：新建单元的 id（含自动分配的）在这里，接着引用它们
  不必再 read 一次、也不用去解析 summary 文本。
- id 省略会自动分配（n1/n2…、e1/e2…）；addEdge 的 from/to 必须是**已存在的节点 id**，
  引用了不存在的节点会**在写盘之前**直接报错并列出已知 id。
- **线的两端可以给绝对点**（fromPoint/toPoint）而不是节点：两端都给点就是一条**独立线** ——
  drawio 里边的两端都可以是自由点，线可以完全不接节点（先画线、之后再拖端点接到节点上也行）。
  只有一端给点就是**悬空端**（另一半接在节点上）。
- **move**：改已有元素的位置。节点给 dx/dy（相对）或 x/y（绝对）；连线只能给 dx/dy
  （平移它自己的折点与自由端点；两端接在节点上时端点由节点决定）。
  一整组一起挪就写 ids:[...] + dx/dy —— 与画布上拖选区是同一套语义（相对位置一点不变）。
  以前没有 move，想把某个节点挪一下就只好删掉重画 —— 那会丢 id、丢边上的端点约束与折点。
- **line**（连线线型，四选一）：straight（直线，不带折点）| sharp（直角折线，缺省）|
  rounded（**圆角折线**：只在折点处倒圆角，rounded=1）| curved（曲线，curved=1）。
  line:"straight" 会**清掉折点**；line:"rounded" 只改样式（折点就是折线的拐角，照旧有用）；
  line:"curved" 在边本来就是两点直连时会补一个弓形中点 ——
  因为 drawio 的曲线是把**现有折线**抹圆，两点时它退化成直线（屏幕上什么都看不出来）。
  与它等价的键级写法是 edgeStyle:"none"（直线）、keys:{rounded:1}（圆角）、keys:{curved:1}（曲线）。
- **fontSize**（字号）：节点标签、独立文字、连线上的文字都吃它（drawio 的 fontSize 键）；
  给 null 就是删键回缺省。例：{op:"setStyle", id:"n1", fontSize:18}。
- **fontColor**（**字色**）：节点标签、独立文字、连线上的文字都吃它（drawio 的 fontColor 键）。
  给调色板名（plain|blue|green|orange|yellow|red|purple|grey）就按画布上"文字换色"那一支取色
  （红 = #b85450，不是填充色），给 #b85450 这类色值就原样写，给 null 删键回缺省黑字。
  例：{op:"setStyle", ids:["n1","n2"], fontColor:"#666666"}；连线上的字同理
  （color 是**线本身**的颜色，fontColor 是**字**的颜色 —— 两件事，别混）。
- **highlight** 只是"让画布选中这几个"给你看，不改文档、也不重排。
- **图层现在已经能操作了**（见上面的 addLayer/setLayer/setLayerProps）：新建一层、
  让新单元进指定层、把已有单元移到别的层、改名/显示/隐藏/锁定。
  隐藏层里的单元照旧读得到 —— 隐藏是**显示状态**，不是删除；用户想看见就提醒他去「图层」菜单打开。
- **容器/分组**：单元带 parent 就说明它在某个容器（分组）里。画布按**绝对位置**显示，
  所以 move 一个容器**不会**带走它的子单元、remove 容器则会把子单元接到还活着的祖先上 ——
  要"整组一起动"就把容器和子单元一起写进 move 的 ids 里。分组本身 AI 还不能新建。
- 糖（shape/style/dash/arrow/color/exit/entry/jettySize/edgeStyle/avoid）由宿主翻译成 drawio 的 style 键，
  **绝不落盘**；keys 用来写任意 drawio 键，值给 null = 删键回默认。
- shape：rect | rounded | **text**（独立文字：无边框无底色的一段字）| stadium | ellipse | diamond | parallelogram | cylinder | document | hexagon | **svg**
  **节点标签与无边框节点**：图片、SVG 内容、视频与独立文字不提供无效描边工具，混合选区描边只作用于有边框的图形和连线。
  用户可独立拖动节点文字到节点内部或外部，双击文字编辑（视频标签也一样）。独立文字节点仍整体拖动。
  节点标签偏移以 labelOffsetX/labelOffsetY（像素）回报，写入原生 mxGeometry 的 mxPoint offset；不要与边标签 labelX/labelY 的相对量混淆。
  偏移随保存、复制、撤销保留；移动节点时文字跟随，导出与适应内容包含节点外的文字。
  **视频节点（旁路媒体包）**：
  - **用户导入入口**：右侧「视频」先选本地文件，再点击画布放置；也可拖入视频或 Ctrl+V 粘贴剪贴板中的视频文件。
    仅支持浏览器可解码的 MP4/WebM/MOV/MKV；视频分块上传。画布须先保存，完整上传后才创建有封面的原视频节点，不创建空占位。
  - addNode {shape:"video", fps?, w?, h?, mediaKind?, mediaClip?, x, y} → 海报 + `*.media/<clip>/`
  - listFrames / getFrame / setFrame / appendFrames / setMediaMeta / importMedia
  - extractFrames（看 stills 计划）/ traceFrame（stills → SVG 帧）
  - exportSvgVideo / exportMediaSource
  - diagram_read 对视频只回 mediaKind/mediaClip/fps/frameCount/duration/hasSource，帧内容用 getFrame
  - **视频复刻交互**：用户选中已导入的普通视频，悬停或点击工具条「绘制复刻」展开设置浮窗。图片绘制同样使用悬停设置卡片。
    复刻结果保留原视频音频与片段起点，播放可静音，SVG 视频包及 WebM 导出携带对应音频。
    开始后视频在后台静默复刻，画布角落显示可收起的进度卡片，不占全屏、不抢焦点，可继续编辑画布、取消任务或失败后按原参数重试；切换画布/页面会取消任务。
    视频工具条复用其他节点的图标在上、文字在下的样式及悬停反馈；绘制与导出使用同一个悬停卡片组件。
    悬浮条保留绘制复刻、预览、抽帧、导出；SVG 节点隐藏绘制入口，导出菜单只展示当前媒体适用的格式。抽帧把当前画面放到右侧：源视频为 PNG，SVG 视频保留当前帧矢量。
    悬停视频节点表面时显示进度条与播放/暂停；停止、逐帧、时间轴、循环和适配等完整控件集中在预览窗口，选区悬浮条不再提供设置封面按钮。
    用户决定完整视频或起止时间范围、任意正数帧率（支持 25/29.97/30/60fps 等）、480/887/1440px 长边、原始尺寸或自定义长边。
    碎片数量可选低/中/高，内部自动匹配每帧 60000/120000/248000 字符预算；保留画风、颜色数和去背景选项。
    不再按 30 秒、360 帧或 640px 截断。面板估算帧数；源视频末尾自动截短，小视频不放大。
    单帧保留 1600 万像素及 16384px 长边的处理边界，超过时明确提示用户选择较小尺寸，不静默降采样。
    浏览器负责解码和分块传输高清 PNG；抽帧与常驻 Worker 池描边流水线重叠，帧按序落盘，仅计算选中碎片档位。图片原有的 887px 校准上限不应用于视频。
    面板显示实际进度和最近完成帧，长 SVG 序列按需预取并最多缓存 48 帧内容。
    原视频保留，全部成功后在右侧新建 SVG 视频节点；取消、失败、切换画布会停止任务，取消/失败清理本次临时媒体。
    失败可重新复刻（从头开始）。完成后可播放、逐帧查看、撤销新增节点，以及导出 SVG 视频包。
    SVG 视频包为 .svgvideo.zip（manifest + SVG 帧 + 封面），不是单文件动态 SVG，不含音轨。逐帧绘制暂不做跨帧稳定处理。
  - **AI 分步处理视频**：先 importMedia，再 extractFrames {from,seconds,fps,width,max} 下发计划；
    返回计划不等于完成，须再次查询 extractFrames 确认 stills 数量及回执，再按实际帧数逐个 traceFrame {id,index,removeBg,…}。
    全部写出后 setMediaMeta {id,kind:"svg",fps,w,h,posterFrame:0}，用 listFrames 核对帧数后 exportSvgVideo。

  要"一段不带框的说明/标题文字"就用 addNode {shape:"text", label:"…", x, y} —— 它就是 drawio 的
  text 形状（样式原样写 text;html=1;whiteSpace=wrap;strokeColor=none;fillColor=none;align=center;verticalAlign=middle;rounded=0）。
  文字元素想换颜色要写 **fontColor**（setStyle {id, keys:{fontColor:"#b85450"}}）：
  它没有填充与描边，往它身上写 fillColor/strokeColor 屏幕上不会有任何变化。
- **svg：节点内容是一段内嵌 SVG 图形**（不是标签文字）。想画流程图词汇表里没有的东西
  （图标、logo、示意图、示意图标、进度环…）就用它 —— 传一段完整的 SVG 标记，节点框里就画它。
  落盘是 drawio 原生的图片形状（shape=image;imageAspect=0;image=data:image/svg+xml,<百分号编码>），
  **drawio 桌面版打开同样能看见这张图**（不是本画布私有的写法）。
  两个用法：
      {op:"addNode", shape:"svg", svg:"<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'><circle cx='12' cy='12' r='9'/></svg>", w:60, h:60, as:"icon"}
      {op:"setStyle", id:"icon", svg:"<svg …>…</svg>"}        // 换一张图（内容整段替换）
      {op:"setStyle", id:"icon", svg:null}                     // 清掉内容 → 回到矩形
  规矩（照做基本不会出错）：
    · **必须同时给 viewBox**（图形按 0 0 W H 的坐标系画，节点框是它的画布）：没有 viewBox 时
      SVG 的坐标系是"像素"，节点放大缩小时图形不会跟着变。缺 xmlns 宿主会自动补（不补的话
      浏览器按 XML 解析会直接丢弃这份图 —— 屏幕上什么都没有，也不报错）。
    · 默认 **imageAspect=0 = 拉伸铺满**节点框；想让图形等比缩放居中就加 keys:{imageAspect:1}
      （那就把 w/h 设成图形的比例，否则四周会留白）。
    · svg 节点通常不带 label（画图不带字）；要图上叠字就照常给 label。普通矩形/面板同样可以省 label。
    · 颜色写在 SVG 自己身上（fill/stroke）—— 配色参数对它是无效的（图片不受 fillColor/strokeColor 影响）。
    · 内容上限 250000 字符；**别放位图/base64**（那是图片不是图形，本画布画不出来）。
    · 改内容只能整段替换（没有"改某一根线"的 op）：先 read 拿回当前 svg，改完整段写回。
- style：调色板名 plain|blue|green|orange|yellow|red|purple|grey，或直接给一段 style 串
- dash：solid|dashed|dotted；arrow：end（单向）|both（双向）|none（无）|start（反向）
- exit/entry：进出侧 n|e|s|w；edgeStyle：orthogonalEdgeStyle|none（none = 直线）
- 自环写成 from === to（与 drawio 一致）；自动布局忽略自环，只摆节点。

## 一段完整的例子
用户说"画一个登录流程，失败回到登录页"：

    diagram_apply({ ops: [
      {op:"addNode", label:"开始",   shape:"stadium", style:"green",  as:"start"},
      {op:"addNode", label:"输入账号", shape:"rect",  style:"blue",   as:"input"},
      {op:"addNode", label:"校验",   shape:"diamond", style:"yellow", as:"check"},
      {op:"addNode", label:"进入首页", shape:"rect",  style:"blue",   as:"home"},
      {op:"addEdge", from:"start", to:"input"},
      {op:"addEdge", from:"input", to:"check"},
      {op:"addEdge", from:"check", to:"home", label:"通过"},
      {op:"addEdge", from:"check", to:"input", label:"失败", dash:"dashed", color:"red"}
    ], layout:"dagre-tb" })

写 as 别名而不是 n1/n2：画布上已经有节点时，自动分配的编号猜不准（错一次就白跑一轮）。

用户说"把这几个（他在画布上选中的）换成绿色"：

    diagram_read()          // 返回里的 selection 就是"这几个"
    diagram_apply({ ops: [{op:"setStyle", ids:[...selection], style:"green"}],
                    expectRevision:"<上一步 read 回来的 revision>" })

## 布局与坐标
- 工作流使用圆角折线；AI 自行根据节点坐标确定水平/垂直通道和 points 折点，不启用自动避障。按流程方向分层、同类对齐、间距一致，检查连线不穿节点或文字。保持用户明确指定的其他线型。
- 简单流程优先使用自动布局：layout 选 dagre-tb（默认，贴合右栏窄高的形状）、dagre-lr（左到右）、
  grid（网格）、none（保留现有坐标，只给没有坐标的新节点补位）。
- 只有"必须摆在某个位置"时才传 x/y（宿主仍会把它吸附到网格）。
- 画布的吸附单位：**节点 10px，连线的折点/自由端点 5px**。手工摆过的画布带 meta.pinned，
  这时**不加 layout 就不会重排**（否则你一改图，人手工排好的版面就被冲掉）；想重排必须显式给 layout。
- **ops 里自带几何时（addNode 给了 x/y、或者有 move）也不会重排** ——
  否则"把 n1 往右挪 40"会被布局立刻冲掉，看起来像工具坏了。
- 显式重排会重算已启用自动避障的旧连线，手动折点和固定连接点保留；节点位置改变后由 AI 复核并按需更新手动折点。新建连线默认手动模式。
- AI 改完的改动**不在用户本地的撤销历史里**（服务端推新版本时客户端会清空它，免得一次 Ctrl+Z
  把 AI 的改动悄悄顶掉）。宿主留了一个**回退栈**（最多 8 步），用户界面「编辑 → 撤销 AI 改动」
  每按一次退一步；read 返回里的 revertSteps 就是"还剩几步"，canRevert 是"还有没有"。
  所以：改了别慌，也别自己"改回去"—— 告诉用户可以连按几次撤销就行。
- **纸张尺寸**（page）：文件里写了就报出来（例如 850 × 1100 = drawio 的 A4 竖版缺省）。
  布局取舍与"要不要给用户导出建议"都该看它 —— 内容远比纸张宽时提醒一句比闷头画更好。

## 想看效果图的时候（diagram_read 带 render:true）
- 传 render:true 就是"让我看一眼这张画布"：正交路由、避让、圆角、文字断线这些
  **只有看图才知道**的东西，几何数据里没有（宿主也不算走线，那是画布算的）。
- 渲染在**浏览器那一半**做，所以只有用户在右侧栏**打开着这张画布**时才有人接活：
  · 画布没打开（宿主知道你打开的是别的文件）→ look.status="not-open"，附原因，别重试；
  · 打开了 → 第一次返回 look.status="pending"（已请求，带 waitedMs），**等几秒再调一次**
    diagram_read（render:true），图就作为图片内容块跟回来了；
  · 挂了请求却一直没人接（25 秒）→ look.status="timeout"，带已等待秒数 —— 这时别再
    傻等，要么请用户先打开画布，要么直接用几何数据。
- **不会在用户的工作区里生成任何文件**：图片存在宿主那边的附件库里（与用户上传图片同一套存储，
  在工作区之外），看完即取走 —— 要再看就再要一次（图不会被反复塞给你）。
- **纯改图不需要它**：几何、样式、图层、纸张从普通 diagram_read 里就有。
  只在判断"这样摆好不好看/线有没有压住节点/别人看不看得懂"时用。
- 看不了的情况会明说：这个部署没挂附件服务、或当前模型没声明图片输入 —— 那时
  look.status 是 unsupported（附原因），照常用几何数据干活即可。
- 那张图是"你请求渲染那一刻"的画面：回执之后你又改过图的话 look.stale 会是 true，
  想看新的就再 render:true 一次。

## style 键与 drawio 同构
文档里存的就是 drawio 的 style 键：fillColor / strokeColor / shape= / rounded= / arcSize= /
dashed / dashPattern / strokeWidth / fontSize / fontColor / edgeStyle / jettySize / orthogonalLoop /
endArrow / startArrow / exitX·exitY·entryX·entryY / libavoidRouting …
**默认值一律省略**（写出来等于多一份噪音），认不出的键原样保留。
边自己的文字存在边的 value 上（diagram_read 里是 label），可以写成多行；
拖动过的边标签位置存在边几何的 x/y/offset 里（沿边比例 + 垂距 + 残余），read 会作为
labelX/labelY 报出来，要改就用 setLabelPos（见上）。

## 画布表示不了、但会原样保留的东西（notes 里会列出来）
多页只显示第 1 页、分组/容器按绝对位置显示、位图/外链图片按矩形显示（内嵌 SVG 会真的画出来）、
HTML 标签按纯文本显示、图层结构原样保留（画布把它们叠在一起显示）、drawio 的独立边标签单元（edgeLabel）只读显示。
**看到 notes 要如实告诉用户**：这份文件在 drawio 里打开时会和这里看到的略有差别。

## 别做这些
- 别直接改 .drawio 文件的文本（用 diagram_apply）：手改会破坏"没动的地方逐字节不变"这条保证。
- 新建优先使用 .drawai；别造其他后缀（.json/.xml/.svg）：画布支持 .drawai 原生 JSON 和 .drawio；其他后缀会"看起来成功、其实没人能打开"。
- 别把图重画一遍来表达"挪一下"：用户手工摆过的位置是有意义的。

## 用户可以这样操作（他问"怎么做"时照这个答）
- 双击节点改文字；双击线段、线段上的文字、或线上的把手，改**线上的文字**
- 按住线上的文字拖动 = 挪文字（有半格吸附，靠近线会贴回线上）；右键「标签居中」放回中点
- 选中连线后：绿/红端点拖到别的节点 = 改接；橙色空心把手拖 = 整段平移；右键有「自动路由」清掉折点
- 空白处左键拖 = 框选（框到节点或线段都算）；Shift 框选是加选；拖选区整体移动（连线形状一起走）
- Ctrl+C/X/V 复制剪切粘贴、Ctrl+Z/Y 撤销重做、Delete 删除；中键拖动平移、滚轮缩放
- 「文件 → 新建画布…/打开…/另存为…」；「视图 → 适应内容」；「整理几何（吸附到格线）」把整张图对齐
- 右键还能改线型/箭头/颜色/顺序（置顶置底）、编辑数据（drawio 用户对象的自定义属性）、删除
