---
name: 'axure-html-task-planner'
description: '将 Axure RP 导出的 HTML 原型目录编译成可供 AI 编码使用的项目上下文包。适用于 Codex 需要读取 Axure HTML export、Axure 原型文件夹、data/document.js、data/sitemap.js，恢复真实页面树，识别页面类型与语义，聚类业务域与主流程，生成需求目录、方案大纲、开发任务计划，以及面向代码生成的 codegen context 的场景。'
---

# Axure RP 原型上下文编译器

## 目标

把 Axure HTML 导出目录编译成一套可复用的项目上下文资产，供 AI 后续写代码时持续引用。重点不是“生成几份分析文档”，而是让 AI 理解：

- 原型真实页面树
- 业务域和主流程
- 页面级功能与规则说明
- 需求清单和开发任务
- 面向编码的项目级与页面级上下文包

## 分析范围硬规则

- `页面原型/`：主分析范围。需求、流程、方案、任务、页面级 codegen context 都只从这里抽主体证据。
- `需求说明/`：补充背景资料。可进入规则说明、术语解释、流程背景，但不直接展开成默认开发任务。
- `更新说明` 或 `更新日志`：版本说明。只作为变更背景，不直接生成业务需求。
- `index.html`、`start.html`、`start_with_pages.html`、`start_c_1.html`：视为 Axure 预览入口页，不计入业务页面。

## 当前正式脚本链

默认使用下面这条正式脚本链，不要回退到旧脚本链：

1. `scripts/extract_axure_structure.py`
2. `scripts/classify_pages.py`
3. `scripts/analyze_page_semantics.py`
4. `scripts/build_domain_map.py`
5. `scripts/build_user_flows.py`
6. `scripts/build_requirements_catalog.py`
7. `scripts/build_solution_outline.py`
8. `scripts/build_implementation_plan.py`
9. `scripts/build_codegen_context.py`

这条链路已经覆盖：

- `data/sitemap.js`
- `data/document.js`
- 黄色说明块抽取
- 需求与任务提示词
- `09-page-contexts/` 页面级上下文报告
- 页面截图与说明卡片裁图

## 历史脚本链说明

下面这些脚本属于历史原型链路，只保留作兼容调试记录或对照参考，不作为默认入口：

- `scripts/extract_pages_tree.py`
- `scripts/analyze_page_content.py`
- `scripts/synthesize_requirements.py`
- `scripts/generate_implementation_plan.py`

如果看到历史调试报告引用这些脚本，按“历史背景”理解，不要把它们当成当前正式产线。

## 核心输出

默认输出根目录为 `<export_dir>/_analysis`。每次正式运行都会新建一个时间戳文件夹：

- `<export_dir>/_analysis/Axure-Task-YYYYMMDD-HHMMSS/`

当前 run 的全部正式产物都放在这个目录里，正式产物如下：

1. `01-axure-structure.md/json`
   恢复真实页面树、目录层级、结构来源和页面计数。
2. `02-page-index.md/json`
   产出页面索引、页面类型、平台、业务域、上下文角色、是否纳入默认分析。
3. `03-page-semantics.md/json/html`
   产出页面语义、关键控件、能力信号、原型说明卡片、开放问题。
4. `04-domain-map.md/json`
   产出业务域聚类、共享能力、关联页面。
5. `05-user-flows.md/json`
   产出主流程、步骤页、流程摘要。
6. `06-requirements-catalog.md/json/html`
   产出需求目录。每条需求都要带证据、规则说明、关联页面、待确认项、Codex 提示词。
7. `07-solution-outline.md/json`
   产出模块方案、共享基础能力、风险与缺口。
8. `08-implementation-plan.md/json/html`
   产出开发任务计划。任务清单必须是 Markdown 表格，并给每个任务附 Codex 提示词。
9. `09-codegen-context.md/json/html`
   产出全项目编码上下文包，并默认同时产出：
   - `09-page-context-index.md/json/html`
   - `09-page-contexts/` 下每个“页面原型”页面的独立 `md/json/html`
   - `_analysis/assets/page-screenshots/`
   - `_analysis/assets/page-annotations/`

## 关键执行原则

### 1. 结构恢复优先于文案推断

- 先恢复真实页面树，再做分类、语义、需求和任务。
- 优先使用 `data/sitemap.js` 或 `data/document.js`。
- 只有结构文件不可用时，才允许退回 HTML fallback。

### 2. 黄色说明块属于强证据

- Axure 黄色说明块视为“原型说明卡片”。
- 卡片信息应优先进入：
  - `03-page-semantics.*`
  - `06-requirements-catalog.*`
  - `08-implementation-plan.*`
  - `09-codegen-context.*`

### 3. 始终区分三类信息

- 显式证据
- 启发式推断
- 待确认项

不要把原型说明、页面标题猜测、或全局导航噪音，直接写成研发事实。

### 4. 任务按能力和工作流拆，不按页面平铺

- 先共享基础能力，后业务域实现。
- 先规则澄清，后编码实现。
- 优先按 `产品/需求`、`前端`、`后端`、`集成`、`测试` 五类工作流拆分。
- 不要把“页面清单”冒充“开发计划”。

### 5. `09` 默认必须覆盖逐页上下文

- 默认运行 `build_codegen_context.py` 时，必须为每个 `页面原型/` 页面生成独立上下文报告。
- 只有显式传入 `--skip-page-reports` 时，才允许跳过 `09-page-contexts/`。
- 如果用户给出目标页或目标流程，再额外使用 `--target` 生成定向上下文。

## 快速执行

默认使用统一入口脚本。它会自动：

- 创建 `Axure-Task-时间戳` 目录
- 顺序执行 9 个阶段
- 让所有阶段共用同一个输出目录
- 更新 `_analysis/latest-run.txt`

推荐命令：

```bash
python scripts/run_full_pipeline.py /path/to/axure-export
```

如果要定向生成目标上下文：

```bash
python scripts/run_full_pipeline.py /path/to/axure-export --target "客户中心列表页"
```

如果只要项目级 `09`，不要逐页页面报告：

```bash
python scripts/run_full_pipeline.py /path/to/axure-export --skip-page-reports
```

## 调试模式

如果你要逐步调试单个阶段，再按下面顺序运行，并给每个脚本传同一个 `--output-dir`。不要在 9 个脚本上各自用默认目录，否则会把同一次分析拆散到不同位置。

```bash
python scripts/extract_axure_structure.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/classify_pages.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/analyze_page_semantics.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/build_domain_map.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/build_user_flows.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/build_requirements_catalog.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/build_solution_outline.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/build_implementation_plan.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
python scripts/build_codegen_context.py /path/to/axure-export --output-dir /path/to/_analysis/Axure-Task-20260522-153000
```

## 交付验收

至少检查下面这些点：

- `02-page-index.json` 中 `is_under_page_prototype=true` 且 `is_preview_entry!=true` 的页面数，和 `09-page-contexts/*.html` 数量一致。
- `06-requirements-catalog.md` 使用 Markdown 表格。
- `08-implementation-plan.md` 使用 Markdown 表格。
- `06` 与 `08` 的 JSON 中都包含 `prompt`。
- `09-codegen-context.*` 和页面级 `09-page-contexts/*` 中都包含推荐提示词。
- 命中黄色说明块的页面，在 `03`、`06`、`08`、`09` 中都能看到规则说明或引用。

## 参考资料

- [references/axure-export-rules.md](references/axure-export-rules.md)
- [references/axure-export-variants.md](references/axure-export-variants.md)
- [references/page-type-rules.md](references/page-type-rules.md)
- [references/capability-detection-rules.md](references/capability-detection-rules.md)
- [references/domain-clustering-rules.md](references/domain-clustering-rules.md)
- [references/planning-rules.md](references/planning-rules.md)
- [references/output-spec.md](references/output-spec.md)
- [references/debugging-guide.md](references/debugging-guide.md)

## 模板与界面资源

- `assets/requirements-template.md`
- `assets/implementation-plan-template.md`
- `assets/solution-outline-template.md`
- `assets/codegen-context-template.md`
- `assets/axure-rp-icon.png`
- `scripts/run_full_pipeline.py`

## 维护约束

- `SKILL.md`、`agents/openai.yaml`、`references/output-spec.md` 必须保持一致。
- 如果改动默认输出行为，先改 `references/output-spec.md`，再改脚本和 `SKILL.md`。
- `debug-report-2026-05-22.md` 是历史调试复盘，不是当前正式说明；正式调试与验收方法以 [references/debugging-guide.md](references/debugging-guide.md) 为准。