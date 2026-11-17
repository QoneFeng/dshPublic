---
name: 'etf-depth-comparison'
description: '对两只 A 股 ETF 做覆盖规模/持仓/收益/风险/产业链的多维量化对比，自动产出结构化 HTML 报告含图表与场景化评估。 适用触发：ETF 横向比较、选基参考、同类基金差异分析。'
---

# A 股 ETF 多维度横向对比

## 能力说明

输入两只 A 股 ETF（支持上交所 `sh` 与深交所 `sz` 代码），自动拉取行情、持仓、净值等数据，产出结构化对比报告，默认以 HTML 形式交付，内含图表与场景化评估。

> **输出约定**：以 `references/etf_compare_template.html` 为骨架 → Agent 直接填充数据生成 HTML；对话仅输出关键发现摘要与文件路径。样式规范遵循 `wb-finance-skill` 的红线（ECharts 加载、`node --check` 语法校验）；视觉采用双色对照体系（ETF-A/ETF-B 分色），详见模板。
>
> **执行顺序**：先加载 `wb-finance-skill` 约束，再调 `westock-data` 取数。若用户提问「哪个更好」，统一改写为适用条件对比与风险提示，禁止输出胜负结论或确定性推荐。

## 适用场景

- 要求对两只 A 股 ETF 做系统对比（如「比较 XXX 和 YYY 两只 ETF」）
- 需要量化分析辅助选基决策（用条件框架输出适配场景，不做确定性荐基）

## 运行环境

| 外部能力 | 作用 |
|----------|------|
| `westock-data` | ETF 搜索、详情、持仓、净值、K 线数据拉取 |
| `wb-finance-skill` | HTML 报告样式规范与 ECharts 图表约束 |

数据查询示例（使用 `westock-data` 官方 CLI，禁止通过 `npx` 等第三方方式调行情）：

```bash
westock-data search 沪深300ETF --type etf
westock-data etf detail sh510300,sz159915
westock-data etf holdings sh510300
westock-data etf holdings sz159915
westock-data kline sh510300 --period day --limit 120
westock-data kline sz159915 --period day --limit 120
# 可选：拉取净值历史
westock-data etf nav sh510300 --start YYYY-MM-DD --end YYYY-MM-DD
```

> `scripts/query.py` 等脚本路径指向对应 skill 目录，本 skill 自身不依赖 clawhub 或第三方 npm 包获取行情。

## 执行流程

### 阶段 A：代码解析与数据采集

**代码解析**：将用户输入的名称或代码 → `westock-data search <名称> --type etf` → 统一成 `sh######` / `sz######` 标准格式。

**数据拉取**：对每只 ETF 分别调用：
- `etf detail` — 获取规模、费率、管理人、跟踪指数等基本信息
- `etf holdings` — 获取前十大重仓及持仓明细
- 日 K 线 — 至少取 120 根（用于手算区间回报）

> 注意：`etf detail` 中部分预计算的回报字段可能存在偏差，**所有区间回报（YTD / 近 1 月 / 近 3 月等）统一用 K 线收盘价自行计算**，不依赖 API 返回值。

### 阶段 B：报告生成

以 `references/etf_compare_template.html` 为模板，按以下模块逐项填入真实数据：

| 序号 | 模块 | 内容要点 |
|------|------|----------|
| 1 | 头部概览 | 标题 + `.tldr` 核心差异摘要（3–5 条） |
| 2 | 关键指标双卡 | 最新价、规模、费率、区间回报 |
| 3 | 基本面信息表 | 管理人、托管人、跟踪指数、成立日 |
| 4 | 重仓持股 | 前十重仓对比表 + ECharts 柱状图 |
| 5 | 走势对比 | 近 60 日归一化走势线图 |
| 6 | 区间回报 | 多周期回报表 + 柱状图 |
| 7 | 风险指标 | 波动率、最大回撤、流动性、涨跌幅限制 |
| 8 | 多维雷达图 | 规模/流动性/分散度/收益/波动/费率/成熟度七维 |
| 9 | 产业链定位 | 持仓行业环节分析（文字） |
| 10 | 综合评估 | 条件→场景对照框架，不输出买入/卖出建议 |
| 11 | 风险提示 | 合规声明 + 数据脚注 |

输出路径：`{OUTPUT_DIR}/<代码1>vs<代码2>_深度对比分析.html`

### 阶段 C：图表约束（ECharts 踩坑清单）

- 默认通过 CDN 加载：`cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js`
- 若 CDN 不可用，改用 head 内联 `echarts.min.js`：此时图表初始化代码必须放在**同一个 `<script>` 标签内**
- **严禁嵌套 `<script>` 标签**（如 `<script><script>…</script>…echarts.init…</script>`），内层 `</script>` 会提前截断外层，导致 `echarts.init` 变为纯文本不执行，图表全部为空
- 交付前对业务 JS（不含整库）执行 `node --check` 语法检查

### 阶段 D：交付

输出一句摘要（3–5 条核心发现）+ 文件路径。涉及图表的报告须 `node --check` 通过后再交付。

## 常见问题与应对

| 问题 | 应对方式 |
|------|----------|
| detail 预计算回报不准 | 用 K 线自行计算区间回报 |
| holdings 为季报数据非当日 PCF | 脚注注明数据日期 |
| 科创板 ETF 涨跌幅 ±20% | 风险表单独标注 |
| 次新 ETF 历史数据短 | 结论中提示样本不足 |

## 合规声明

报告末尾必须包含：「以上分析基于公开数据整理，仅供参考，不构成投资建议。基金过往业绩不代表未来表现。」禁止输出确定性买入/卖出或「哪个更好」结论，统一使用适用条件与风险对照。

## 文件索引

| 用途 | 路径 |
|------|------|
| HTML 报告模板 | `references/etf_compare_template.html` |
| 常见 ETF 代码速查 | `references/etf_code_reference.md` |
| HTML 样式规范 | `wb-finance-skill/references/html-report-style.md` |