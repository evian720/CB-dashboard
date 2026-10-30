# Luca Research（luca-research.com）技术与数据规格说明书 · 中文版

> 用途：本文档由对 https://luca-research.com/ 的全量逆向遍历产出，可直接作为 Prompt 输入给 LLM 用于 1:1 重建该网站的数据层与前后端。
> 站点主题：A 股可转债 / 可交债 / 优先股 / 固收+基金 / ETF / 固收+策略的研究数据看板。
> 逆向时间：2026-10-30，数据截至日：2026-10-29。

---

## 1. 网站全局架构与导航树

### 1.1 技术栈与架构（逆向结论）

- **前端**：React 18 SPA（Vite 构建，路由为客户端路由；JS chunk 按页面懒加载，如 `assets/CbBoard-KJi2ArGM.js`）。UI 组件库为 shadcn/ui + Radix（`data-slot=popover-trigger`、`role=tab` 等 Radix 行为特征）。图表为 ECharts（`<div>` 容器 + canvas 渲染，图例/坐标轴结构与 ECharts 一致）。支持暗色/浅色主题切换（class 切换，顶栏按钮「切换为浅色板块」）。
- **数据层（关键架构特征）**：除鉴权外**没有任何动态后端 API**。所有行情/研究数据以**构建期生成的静态 JSON** 形式部署在 `/data/**.json`，由前端按页面懒加载 fetch。数据文件普遍带 `updated`（数据截至日）字段。
- **鉴权**：唯一动态 API 为 `/api/auth/login`（POST 登录）与 `/api/auth/me`（GET 会话校验，每页加载时调用）。登录页为 `/`（未登录时渲染，含「登录/注册」两个 Tab；注册需管理员审核）。全站内容仅授权用户可见，未登录访问任意路由重定向到登录页。
- **源码组件结构（来自构建产物中的 `code-path` 属性）**：`src/pages/*.tsx`（页面）、`src/sections/*.tsx`（业务区块，如 CbDashboard、CbChipBond、SiteHeader、SiteFooter）、`src/components/*.tsx`（通用组件：AnchorNav、ChartRangePicker、StrategyNavChart、CandleChart、FeedbackWidget、GuideWidget、ui/table 等）。
- **全局悬浮件**：右下角「留言反馈」（FeedbackWidget）与「站点导览」（GuideWidget，自然语言问答助手，提供直达链接）。
- **页脚**：京ICP备2026063046号（链接 beian.miit.gov.cn）。

### 1.2 站点地图（全部路由）

```
/                         首页 = 登录页(未登录) / 每日观察 Daily Brief(已登录)
/cb                       可转债研究主页（6 个模块卡片入口）
  /cb/segment             转债分域情况（12 维度）
  /cb/clause              转债条款跟踪（5 个 Tab）
  /cb/chips               转债及其正股筹码（3 个 Tab）
  /cb/board               转债个券看板（48 列全量表格）
  /cb/quant               转债量化策略（3 策略）
  /cb/market              → 重定向 /cb/market/overview
    /cb/market/overview   市场概况（规模/指数/供给）
    /cb/market/valuation  市场估值（估值指标/百元溢价率/换手/异常标的）
    /cb/market/greeks     希腊字母与归因（+月度复盘文章）
    /cb/market/holders    投资者情况（转债ETF + 交易所投资者结构）
  /cb/bond/:code          个券详情页（如 /cb/bond/110081.SH）
/eb                       可交债研究主页（2 个模块卡片）
  /eb/board               可交债综合看板（485 只全量表）
  /eb/market              可交债市场跟踪（私募/公募 2 个 Tab）
  /eb/notes               可交债研究笔记（仅本地环境提供内容）
/ps                       优先股研究主页（2 个已上线模块 + 1 筹备中）
  /ps/board               优先股综合看板（29 只）
  /ps/market              优先股市场跟踪
/fund                     基金研究主页（3 组 10 个子模块 + 2 筹备中）
  /fund/hf                固收+基金高频跟踪看板（2141 只）
  /fund/market            固收+基金市场整体观测
  /fund/shelf             固收+管理人货架分布（143 家）
  /fund/strategy          固收+基金策略（指数/增强FOF/画线组合）
  /fund/pb                纯债基金跟踪看板（1674 只）
  /fund/pb-duration       纯债基金久期跟踪（Kalman）
  /fund/pb-shelf          纯债管理人货架分布（157 家）
  /fund/lowvol-strategy   纯债基金指数（基准+增强）
  /fund/bond-market       债券型基金市场综合看板
  /fund/pension           养老金产品跟踪（RBSA）
/etf                      ETF 策略研究主页（4 个已上线 + 1 筹备中）
  /etf/board              ETF 个券看板（1698 只）
  /etf/scale              ETF 市场结构看板
  /etf/pb                 纯债 ETF 综合看板（51 只）
  /etf/stock-etf          股票 ETF 策略（3 动量策略）
/strategy                 固收+策略研究主页（2 个模块）
  /strategy/income-equity 生息固收+权益策略（红利增强选股）
  /strategy/multi-asset   多资产配置模型（股债两资产 44 组参数）
/about                    关于本站（网站更新日志）
```

### 1.3 全局控制器（Global State / Filters）

| 控制器 | 作用域 | 默认值 | 可选项 |
|---|---|---|---|
| 主题切换 | 全站 | 深色 | 深/浅（顶栏按钮，持久化） |
| 登录态 | 全站 | — | `/api/auth/me` 校验；失败重定向登录页 |
| **ChartRangePicker（时间窗）** | 每个图表组件局部，但全站统一 UI | `全部` | `全部 / 近5年 / 近3年 / 近1年 / 今年以来 / 自定义起点`（自定义为 date input，格式 YYYY-MM-DD） |
| **下载按钮** | 每个图表局部 | — | 导出当前图 CSV（部分表为「月度数据 CSV」「规模明细 CSV」） |
| 权重口径 | /cb/segment、/cb/market（指数） | 等权 | 等权 / 市值加权（segment 页顶栏按钮组） |
| 暗/浅配色联动图表 | 全部 ECharts | 跟随主题 | 图表色板随主题切换 |

> 没有跨页面的全局时间筛选或资产切换器；时间窗是**组件级**状态。模块内二级导航（如 /cb/market 的 4 个子 Tab）为页面级 Tab 而非全局。

### 1.4 通用组件规格（全站复用）

1. **AnchorNav（本页导航）**：左侧/顶部 sticky 导航，列出页内各 Section 标题按钮，点击平滑滚动定位。
2. **ChartRangePicker**：按钮组（全部/近5年/近3年/近1年/今年以来）+「自定义起点」日期输入框 + 「下载」按钮。改变窗口后：净值类图在窗口内重新基点化（首值=100 或 =1000），指标表按窗口重算（前端计算）。
3. **KpiCard**：label + value 的小卡（如「基金总数 2141 只」）。
4. **DataTable**：shadcn Table，支持列排序（点击列头 asc/desc 切换）、固定表头、行链接（代码列跳详情页）、分页（如 50/页，43 页）。
5. **Tooltip**：ECharts 默认十字准线 + 系列名/日期/值。

---

## 2. 逐页深度拆解

### 2.0 页面：首页 Daily Brief（URL: `/`）

**功能定位**：登录门 + 每日晨报（3-5 条市场要点 + 分级风险提示），每个交易日更新，读数取自站内各板块当日数据。

**组件清单**：
- **Hero 动画**：单轮播放——线稿键盘动画 → 打字机打出 "Luca Luo · Daily Brief" → 停留。
- **Card_DailyBrief**：`日期`（如 2026-10-29）+「最近一期」标签 + 摘要卡片列表。每张卡片：`tag`（板块标签：转债/基金/...）+ `text`（300 字内市场要点摘要）+ 链接 `link`（直达对应模块）。
- **Card_RiskSignals**：分级风险提示列表（level: high/middle/low + text + link），无风险时空置。
- **Footer**：免责声明「内容仅供研究参考，不构成投资建议」。

**数据源**：`GET /data/daily_brief.json`
```json
{
  "date": "2026-10-29",
  "items": [{ "tag": "转债", "text": "中证转债跌 1.13% 收 1600.1…", "link": "/cb/market/overview" }],
  "risks": [],
  "auto": {
    "cbDate": "2026-10-29", "cbN": 319, "cbMv": 5967.95, "cbMvChgPct": -1.12,
    "cbpMed": 59.34, "ytmMed": -3.53, "prem100": 41.78, "prem100Pctile": 93.1, "cbpMedPctile": 97.5,
    "abnPct": 1.88, "turnMed": 4.21, "zzcb": 1600.1, "zzcbChg": -1.13,
    "cbMvIdx": 1616.3, "cbMvIdxChg": -1.09,
    "callExecYtd": 89, "callNoexecYtd": 107, "resetExecYtd": 72, "resetNoexecYtd": 300,
    "etfDate": "2026-10-29", "etfScale": 50414.7, "etfScaleChgW": 1049.2, "etfBondScaleChgW": 269.8,
    "riskSignals": [{ "level": "high", "link": "/cb/market", "text": "百元溢价率 41.78% ，处近3年 93.1% 分位" }]
  }
}
```
**auto 字段语义**：cbN=转债只数；cbMv=转债市值(亿)；cbpMed=转股溢价率中位数(%)；ytmMed=YTM中位数(%)；prem100=百元溢价率(%)；prem100Pctile=百元溢价率近3年分位；cbpMedPctile=溢价率分位；abnPct=异常标的占比(%)；turnMed=换手率中位数(%)；zzcb=中证转债收盘；cbMvIdx=转债市值加权指数点位；callExecYtd/callNoexecYtd=年内强赎执行/不执行公告数；resetExecYtd/resetNoexecYtd=年内下修执行/放弃数；etfScale=ETF总规模(亿)；etfScaleChgW=周变化(亿)。

---

### 2.1 模块：可转债（`/cb`）

#### 2.1.1 /cb 研究主页
6 张模块卡片（状态徽章「已上线」+ 标题 h2 + 一句话描述 + 「进入模块」）：segment / clause / chips / board / quant / market（描述见站点地图）。

#### 2.1.2 页面：转债市场整体观察 · 市场概况（URL: `/cb/market/overview`）

**功能定位**：转债市场规模与只数、指数表现与供给结构的长期跟踪。

**组件清单**（自上而下）：

1. **Chart_CBScale（转债整体规模）** 双图
   - 图 A：双轴折线（左轴 0~12000 亿元）：`存量余额(亿元)`、`存量市值(亿元)`；顶部摘要「最新：余额 4738 亿 · 市值 5968 亿 · 319 只」。
   - 图 B：单线 `存量只数`（左轴 0~600 只）。
   - X 轴：月度（2004-10 起，2005-01/2020-04… 刻度）。
   - 控件：ChartRangePicker + 下载。
2. **Chart_CBIndex（转债/正股指数）**
   - 副标题「（月度换仓，基点 1000）」；摘要「最新：转债等权 2272 · 转债市值加权 1616 · 正股等权 1707 · 正股市值加权 1462」。
   - 5 条折线（可点图例隐藏）：`中证转债 / 正股市值加权 / 正股等权 / 转债市值加权 / 转债等权`。
   - **Table_IndexPerf（指数区间绩效表）**：列 = 指数 | 区间累计收益率 | 区间年化收益率 | 年化波动率 | 最大回撤；5 行（转债等权 127.23%/9.77%/12.62%/-21.34% 等）。
   - 口径注（原文）：「转债等权/市值加权：公募转债（余额>3000万，剔除已触发强赎/摘牌）月度换仓，票息复权（AdjPrice）；正股等权/市值加权：同期持仓 1:1 对应的正股组合（同一正股多只转债占多份），价格取后复权收盘；中证转债为价格指数对照。净值在所选区间内按起点重新基点化 1000，指标按所选区间计算（年化按 243 个交易日）。」
3. **Section_CBSupply（转债供给结构）**
   - 图 A **转债新发**：柱（左轴 发行规模(亿)）+ 线（右轴 新发只数）。
   - 图 B **转债退市**：柱堆叠（转股退市规模(亿) / 其他退市规模(亿)）+ 线（右轴 退市只数）。
   - 图 C **净供给**：柱 = 当月净供给（红=净增/绿=净减）+ 线 = 近12个月累计(亿)。公式「新发规模 − 退市规模（发行规模口径）」。
   - **Table_YearlySupply（年度汇总表）**：列 = 年份 | 新发只数 | 新发规模(亿) | 转股退市只数 | 转股退市规模(亿) | 其他退市只数 | 其他退市规模(亿) | 净供给只数 | 净供给规模(亿) | 转股消化(亿) | 兑付消化(亿)（2010-2026 全量 17 行）。注：「转股消化 = 退市债存续期已转股的规模；兑付消化 = 终态现金兑付的规模（发行规模 = 转股消化 + 兑付消化）」。
   - 控件：时间窗 + 「月度数据 CSV」「CSV」。

**数据源**：
- `GET /data/cb_market.json` → `{updated, daily:{cal:string[2397] (YYYYMMDD, 20041019~), rows:number[2397][28]}, holder:{sse,szse}}`。**rows 为 28 列紧凑数组**，列定义（已逐列验证）：
  ```
  [0] 存量只数 N
  [1] 存量余额(亿)
  [2] 存量市值(亿)
  [3] 转换价值·中位数   [4] 转换价值·平均数   [5] 转换价值·市值加权
  [6] 债底价值·中位数   [7] 债底价值·平均数   [8] 债底价值·市值加权
  [9] 剩余期限·中位数(年) [10] 剩余期限·平均数 [11] 剩余期限·市值加权
  [12] YTM·中位数(%)    [13] YTM·平均数      [14] YTM·市值加权
  [15] 转股溢价率·中位数(%) [16] 转股溢价率·平均数 [17] 转股溢价率·市值加权
  [18] 债底溢价率·中位数(%) [19] 债底溢价率·平均数 [20] 债底溢价率·市值加权
  [21] 百元溢价率(%)
  [22] 百元溢价率回归样本数
  [23] 异常标的个数
  [24] 异常标的占比(%)
  [25] 换手率·中位数(%)  [26] 换手率·平均数  [27] 换手率·市值加权
  ```
  最新行示例：`[319, 4737.7, 5967.95, 81.32, 91.43, 94.79, 106.55, 105.57, 106.6, 2.69, 3, 2.59, -3.53, -9.65, -9.06, 59.34, 73.55, 63.56, 19.83, 36.37, 30.46, 41.78, 293, 6, 1.88, 4.21, 16.14, 8.64]`
- `GET /data/cb_idx.json` → `{updated, cal:string[2140] (YYYYMMDD, 20171229~), series:{CB_EQ:[], CB_MV:[], STK_EQ:[], STK_MV:[], ZZCB:[]}}`（每日净值点位，月度换仓基点 1000）。
- `GET /data/cb_supply.json` → `{updated, monthly:[202]{m:"2010-01", issue_n, issue_sz, conv_n, conv_sz, other_n, other_sz, dig_conv, dig_cash}, yearly:[17]{y, issue_n, issue_sz, conv_n, conv_sz, other_n, other_sz, dig_conv, dig_cash}}`。

#### 2.1.3 页面：市场估值情况（URL: `/cb/market/valuation`）

**组件清单**：

1. **Tabs_ValuationMetrics（转债估值指标）**：6 个 Tab——`转换价值 / 债底价值 / 剩余期限 / YTM / 转股溢价率 / 债底溢价率`。每个 Tab 内：3 个 KPI（中位数 / 平均数 / 市值加权）+ 折线图（3 条线同名）。字段单位：转换价值/债底价值=元；剩余期限=年；YTM/两溢价率=%。数据即 `cb_market.json` rows 对应 3 列。
2. **Chart_Prem100（百元溢价率，回归拟合）**：单线（%），摘要「最新：41.78% · 回归样本 293 只」。
   - **公式（站方原文）**：「每日截面回归 y = a·e^(-b·x) + c：y 为转股溢价率、x 为转换价值/100，取 x=1（转换价值 100 元）处的拟合值；有效样本为未公告强赎且剩余期限 0.5 年以上的转债。」
3. **Chart_Turnover（换手率）**：3 线（中位数/平均数/市值加权，%）。公式「换手率 = 每日成交金额/总市值；市值加权以个券总市值为权重」。
4. **Chart_Abnormal（异常标的）**：标题「异常标的（价格≥130 且 转股溢价率≥30% 且 换手率≥100%）」；柱（个数，左轴）+ 线（占比%，右轴）；摘要「最新：6 只 · 占比 1.88%」。

#### 2.1.4 页面：转债希腊字母与归因（URL: `/cb/market/greeks`）

**页头口径（原文）**：「退市调整 BS 模型；贴现率为纯债价值反推的个券债底 YTM；退市线=max(1/股价, 3亿/总市值)×平价；预期收益率=delta·s·rf+½·gamma·(s·σ)²+theta 的年化口径；归因为全重定价（full revaluation）口径」。

**组件清单**：

1. **Chart_MarketDelta（全市场 Delta，无量纲）**：3 线（中位数/平均数/市值加权，左轴 0~1）+ 右轴样本只数柱。摘要「最新：中位数 0.681 · 平均数 0.655 · 市值加权 0.647 · 308 只」。
2. **Chart_ExpRet（转债预期收益率，% 年化）**：3 个子 Tab：
   - 三条线：中位数/市值加权/平均数（%）。
   - 分项分解：堆叠图「预期收益率 = delta漂移 + gamma波动 + theta损耗」（delta/gamma 向上堆叠、theta 向下堆叠，浅色线=合计平均数）。
   - 波动率对照：隐波中位数 vs 正股 120 日历史波动率中位数（%）。
3. **Chart_AboveYTM（预期收益高于债底 YTM 占比）**：线（占比%，左轴）+ 柱（只数，右轴）。公式「占比 = 当日预期收益率高于个券债底 YTM 的转债只数 / 当日有效样本只数」。
4. **Table_AboveYTMLatest（最新一期命中标的）**：列 = 代码(链接) | 名称 | 收盘价 | 平价 | 预期收益率% | 债底YTM% | 差值%；按差值降序，共 32 行。
5. **Chart_Attribution（希腊字母归因，月度，% 全重定价）**：堆叠柱（delta/gamma/rho/theta/vega/残差，正贡献在 0 轴上堆叠、负贡献在下）+ 浅色线=当月实际收益。
6. **Accordion_Explain（指标解释与归因方法，折叠面板）**：完整模型文档（BS 公式、五因子定义表、长期参照 2017-02~2026-09 共 116 个月：实际 +0.69 = delta +0.49 + gamma +0.25 + theta −0.04 + vega −0.02 + rho +0.06 + 残差 −0.05）。
7. **Section_MonthlyReview（月度复盘文章）**：`combobox` 选择期数（index 由 `/data/cb_reports/index.json` 提供，2021-01~2026-10 共 70+ 期，draft=true 显示「更新中」徽章）；正文为结构化长文（市场总览 / 收益归因与周内节奏 / 估值与情绪 / 分域结构 / 个券具象化 / 供给与条款 / 需求与资金面）+ 归因表（区间 实际收益 delta gamma theta vega rho 残差 样本数，含 MTD 合计行）。

**核心计算逻辑（站方口径全文归纳）**：
- **理论价**：`V = 存活概率 × [ 平价s·N(d1) + 到期赎回价X·e^(−r·t)·(1 − N(d2)) ]`；s=转股价值；t=剩余期限；X=到期赎回价；r=债底YTM（贴现率）；σ=隐含波动率（市场价格反解）；存活概率=正股在剩余期限内不跌破退市线（退市线=max(1/股价, 3亿/总市值)×平价）的概率。
- **Delta**：`Δ = ∂V/∂s = N(d1)·存活概率 + 退市项修正`。
- **预期收益率**：`E[ret] = ( Δ·s·r_f + ½·Γ·(s·σ_his)² + Θ ) / P`，其中 r_f=无风险利率（站方假设正股年化期望收益=4%）；σ_his=正股历史波动率。
- **月度五因子归因（v2 全重定价）**：`月收益 = delta项 + gamma项 + theta项 + vega项 + rho项 + 残差`；每步将新参数（先平价 s、再隐波 σ、再期限 t、最后利率 r）代回模型完整重新定价，价差即因子贡献。

**数据源**：`GET /data/cb_greeks.json`
```jsonc
{
  "updated": "2026-10-29",
  "delta":    [{ "d": "2017-01-03", "median": 0.669, "mean": 0.687, "mv": 0.69, "n": 15 }, ...],  // 2383 日
  "exp_ret":  [{ "d": "...", "median": 0.23, "mean": -0.147, "mv": 0.697, "n": 298,
                 "iv": 68.725, "vol_his": 48.383,        // 隐波中位/历史波动中位(%)
                 "d_term": 1.624, "g_term": 1.411, "t_term": -3.096 }],  // 三分项(%)
  "above_ytm": { "series": [{"d","n","pct"}], "latest_date": "...",
                 "latest": [{"code","name","close","conv_v","exp_ret","ytm","gap"}] },   // 按gap降序
  "attribution": [{ "m": "2017-02", "n": 15, "ret_actual": 0.341, "delta": 2.045, "gamma": 0.061,
                    "theta": -0.016, "vega": -1.82, "rho": 0, "residual": 0.071 }, ...]  // 117 月
}
```
报告：`/data/cb_reports/index.json`（months 索引）、`/data/cb_reports/{YYYY-MM}.json`（title+body）、`/data/cb_reports/explain.json`。

#### 2.1.5 页面：投资者情况（URL: `/cb/market/holders`）

**组件清单**：

1. **Section_CBETF（转债 ETF）**：固定两只——博时可转债ETF（511380.SH）、海富通可转债ETF（511180.SH）。摘要「最新合计规模 704.8 亿元 · 占转债(含公募EB)市场 11.43%」。
   - **规模与份额**：子 Tab「市值规模 / 份额」；3 线（博时/海富通/合计，亿元）。公式「规模 = 总份额 × 收盘价；份额为交易所披露总份额（变动日记录，交易日间前向填充）」。
   - **占转债市场规模比例**：3 线（%）。公式「分母 = 转债总市值 + 公募EB总市值（两只 ETF 持仓均含公募可交债）；公募EB 市值 = 收盘价(前向填充) × 存续面值」。
   - **资金流**：子 Tab「净申购估算 / 主力净流入」；柱（两只各自日净申购，亿元，堆叠）+ 线（合计近 20 日累计，右轴）。公式「净申购估算 = 当日总份额变动 × 收盘价」。
2. **Section_HolderStructure（交易所投资者结构）**：Tab 上交所/深交所；子 Tab「持有结构（面值亿元）/ 成交结构（成交金额亿元）」；时间范围 combobox（近3年等）。
   - 图：百分比堆叠面积图（0~100%），每色一类投资者。
   - **Table_LatestMonth（最近月份明细）**：投资者类型 | 面值(亿元) | 占比%（按面值降序）。
   - SSE 17 类投资者：公募基金、企业年金、自然人投资者、证券公司自营、一般机构、私募基金、保险机构、证券公司资管、基金管理公司资产管理计划、社会保障基金、QFII、RQFII、信托机构、银行理财、其他金融机构、银行自营、财务公司、沪股通投资者。
   - SZSE 14 类：基金、企业年金、自然人投资者、一般机构、私募基金、券商自营、保险机构、其他专业机构、券商集合理财、基金专户、社保基金、信托机构、RQFII、QFII。

**数据源**：`GET /data/cb_etf.json`
```jsonc
{
  "updated": "2026-10-29",
  "etfs": [{ "code": "511380.SH", "name": "博时中证可转债及可交换债券…基金", "short": "博时可转债ETF" }, {…511180.SH}],
  "cal": ["20200407", …],            // 1591 日
  "cbMv": [4426.31, …],              // 转债总市值(亿) 同长度
  "ebMv": [1081.03, …],              // 公募EB市值(亿)
  "series": { "511380.SH": { "shares":[亿份], "mv":[亿], "flow":[亿,首日null], "mflow":[主力净流入亿], "amt":[成交额亿] }, "511180.SH": {…} }
}
```
持有人矩阵在 `cb_market.json → holder`：
```jsonc
"holder": {
  "sse": { "months": {"0":"2016-07", …}, "investors": {"0":"公募基金", …(17)},
           "hold": {"0": {"0":125.74, …(月→亿, null=未披露)}, …(投资者→月)},   // 面值(亿元)
           "trade": {…同构}, "holdUnit": "面值(亿元)", "tradeUnit": "成交金额(亿元)" },
  "szse": { "months": {"0":"2019-05",…}, "investors": {…14}, "hold": {…市值(亿元)}, "trade": {…} }
}
```

#### 2.1.6 页面：转债分域（URL: `/cb/segment`）

**功能定位**：按 12 个维度对转债分组，观察各分域指数表现、成分数量/市值与当前持仓。

**全局控件**：权重口径（等权/市值加权，顶部按钮组）+ 维度 Tab×12 + ChartRangePicker。

**12 维度完整定义（站方原文，来自 dims 元数据）**：

| key | 标签 | 分组规则 |
|---|---|---|
| style | 股债性 | 转换价值/纯债价值：<0.8 债型；0.8~1.2 平衡型；>1.2 股型 |
| price | 价格 | 收盘价：<110 低价；110~135 中价；>135 高价 |
| industry | 申万行业 | 正股申万一级行业（2021 版） |
| major | 大类风格 | 申万一级行业归并为 7 大类（周期品/装备制造/科技/必选消费/可选消费/稳定/大金融） |
| maturity | 剩余期限 | 六组互斥全覆盖：新券=上市≤2月；次新券=未进转股期；普券=剩余>2年；老券=1~2年；临期券=1月~1年；末日轮=已公告强赎或剩余≤1月 |
| rating | 债项评级 | 高=AAA/AA+；中=AA/AA-；低=A+/A；弱=A-及以下（含违约）；无评级不入组（评级日时点化） |
| size | 发行规模 | 发行规模（静态）：≥50亿 大；20~50亿 中大；5~20亿 中小；<5亿 小 |
| stk_mv | 正股市值 | 换仓日正股总市值：≥500亿 大；200~500亿 中大；100~200亿 中等；50~100亿 中小；<50亿 小 |
| fund_hold | 基金持有 | 公募持有市值/该券总市值（季报/中报/年报，A/C 去重，公告日时点化）：≥50% 基金重仓；30~50% 相对重仓；10~30% 相对低仓；<10% 基金低仓；未进转股期单独「未转股期」组；转股期内无披露视为 0（低仓组） |
| doublelow | 双低 | 换仓日截面：价格名次+溢价率名次=双低值，五等分 Q1(最双低)~Q5 |
| stk_vol | 正股波动率 | 正股过去 120 日日收益率标准差（后复权）五等分 Q1(最低)~Q5 |
| stk_mom | 正股动量 | 正股过去 55 日累计收益率（后复权）五等分 Q1(最弱)~Q5(最强) |

**每个维度 Tab 的组件（布局统一）**：
1. **Table_SegmentPerf（区间表现比较）**：行 = 各分组 + 「全样本基准」；列组 = 近1周(起始日) / 近1月 / 本季度以来 / 今年以来 / 近3年，每组 3 子列：收益% / 回撤% / 波动%（年化）。首两列 = 最新点位（基点1000）、成分数。表头第二行为动态起始日期（如「近1周 10-21 起」）。
2. **Chart_SegmentNav（净值走势）**：各分组净值线 + 虚线全样本基准；窗口起点归一 100；顶部列出「基期 2016-12-30 = 100 · 区间收益率：全样本 +99.5% …」。
3. **Chart_SegmentCount（成分券数量变化）**：堆叠累积柱/面积（只），虚线=全样本总数。
4. **Chart_SegmentMv（成分券市值变化）**：堆叠累积（亿元），虚线=全样本合计。
5. **Table_CurrentHoldings（当前持仓明细）**：combobox 选分组；说明行「换仓日 2026-09-30 · 成分 124 只 · 价格/规模截至 2026-10-29 · 组内等权」；列 = 转债代码(链接) | 名称 | 收盘价 | 股债性比 | 剩余规模(亿) | 申万行业 | 大类风格。

**换仓口径**：月末换仓、基点 1000（页头原文「各类型转债基本表现：月末换仓、基点 1000」）。

**数据源**：
- `GET /data/cb_segment_idx.json` → `{updated, dims:[12]{key,label,desc}, groups:{<dim>:{<组名或数字索引>:{…}}}, bench:"全样本", cal:[2384](YYYYMMDD, 20161230~)}`。groups 序列对象键为 `s`（净值）、`c`（只数）、`n`、`mv`（市值）。
- `GET /data/cb_segment_idx_style.json` → `{series:{全样本/债型/平衡型/股型:{s,c,n,mv}}}`（股债性维度的完整序列）。
- `GET /data/cb_segment_hold.json` → `{updated, holds:{<组名>:{rebal:"2026-09-30", n:124, items:[[code,name,close,style_ratio,size_yi,industry,major], …]}}}`（组名为中文标签，如「债型」「Q1(最双低)」「波动率Q5(最高)」「基金重仓(≥50%)」等 60+ 组）。

#### 2.1.7 页面：转债条款跟踪（URL: `/cb/clause`）

**功能定位**：存续个券强赎/下修/回售条款的逐日触发进度，按距触发天数排序。

**Tab 1 · 有条件强赎**：
- **Chart_CallYearly**：逐年柱（红=公告赎回执行 / 灰=公告不赎回），2013~2026。
- **Table_Call**（319 只）：搜索框（代码/名称/正股）+ 状态 combobox（全部状态）。列 = 代码(链接) | 名称 | 正股 | 转债收盘 | 转股价值 | 转股价 | 剩余期限(如 5.1年) | 触发进度(如 26/15，已达天数/需达天数) | 状态 | 豁免/事件。状态枚举：`已触发 / 计数中 / 豁免期` 等。

**Tab 2 · 转股价下修**：
- **Chart_ResetYearly**：逐年柱（蓝=提议下修 / 灰=公告不下修）。
- **Table_Reset**（319 只）：列同上，但「最近下修 / 事件」列（如「已提请下修，未公告下修价格 / 下修至 6.18（2025-03-27）」）。状态含 `已提议待生效 / 已触发 / 计数中`；触发进度可为「暂停计数」。

**Tab 3 · 有条件回售**：
- **Table_Put**（319 只）：列 = …| 触发进度（如「连续167天」）| 状态（已满足条件）| 备注。含 `days_to`（距回售期天数）。

**Tab 4 · 年度统计**：
- 「历年强赎公告与下修生效数量及标的情况（公告/生效时剩余期限、此前累计条款执行情况）」；年度柱图 + 分年清单（combobox 选年）：`强赎（89）/下修（67）` 两组表格，列 = 代码 | 名称 | 公告日 | 距摘牌(如 33天) | 此前条款执行情况（此前触发N次 / 不赎回N次 / 已强赎N次）。

**Tab 5 · 事件收益**：
- 口径：「公告事件后 5/10/20 个交易日的收益统计（基准日=公告日，含公告日当天涨跌；转债超额基准=中证转债指数，正股超额基准=中证800全收益）」。
- 控件：年度/季度切换；指标切换（转债收益/转债超额/正股收益/正股超额）；窗口（5日/10日/20日）；4 个事件小图（下修-执行（提议下修）/下修-不执行（公告不下修）/赎回-执行（公告赎回）/赎回-不执行（公告不赎回）的年度均值柱状）。
- **Table_EventStats**：列 = 年份 | 下修样本数(合计/执行/不执行) | 赎回样本数(合计/执行/不执行) | 下修-执行(均值%/胜率%/赔率) | 下修-不执行(…) | 赎回-执行(…) | 赎回-不执行(…)。2013~2026 全量。

**数据源**：
- `GET /data/clause_board.json` → `{updated, call:[319], reset:[319], put:[319]}`，每条记录字段：
  ```
  code, name, stk_name, cb_close, convv(转股价值), conv_price, ptm(剩余期限年),
  ratio(触发比例%，强赎130/下修85/回售70), window(计数窗口天,30), need(需达天数,15或30),
  cnt(已计数天，null=不适用), gap(距触发天数), status(triggered/counting/proposed/exempt…),
  exempt_end(豁免截止), last_event(最近事件日), last_reset:{date,price}(下修记录), note, days_to(回售)
  ```
- `GET /data/clause_events.json` → `{updated, counts:{yearly:{call:[{period,exec,noexec}], reset:[…]}, quarterly:{…}}, stats:[{period, event:"下修"|"赎回", executed:bool, n, metrics:{cb_ret_5d/10d/20d, cb_excess_5d/10d/20d, stk_ret_5d/10d/20d, stk_excess_5d/10d/20d: {mean, win, payoff, n}}}]}`。

#### 2.1.8 页面：可转债筹码（URL: `/cb/chips`）

**Tab 1 · 个券筹码**：
- **ComboBox_BondPicker**：cmdk 搜索（placeholder「选择个券（代码 / 名称搜索）…」，选项格式 `110081.SH 闻泰转债 电子`，来源 chips_index）。
- 选中后：头部卡（代码/名称/行业/存续徽章/正股代码）。
- **Chart_PriceVolume（价格走势 OHLC + 成交量 + 筹码加权价格）**：K 线 + 成交量柱 + 琥珀色筹码加权价格线。
- **Chart_ChipFactor（筹码因子 %）**：单线。**公式：筹码因子 = 收盘价 / 筹码加权价格 − 1**（如 2026-10-29：-12.75%）。
- **Chart_ChipDist（筹码分布）**：日期选择（默认最新）；横向直方分布，配色：低于收盘价=浮盈筹码 / 高于收盘价=套牢筹码；标注「收盘 90.27 · 加权 103.47 · 浮盈 1.5%」（浮盈比例 = 低于收盘价的筹码占比）。

**Tab 2 · 市场筹码**：
- **Chart_MarketFactor（全市场筹码因子 %）**：红线市值加权 / 灰线等权（2014-12 起）。
- **Chart_Underwater（收盘价低于成本价的筹码占比，市值加权）**：单线 0~100%。
- **Chart_IndustryFactor（行业筹码因子最新截面）**：横向条形（申万一级 29 行业，-18%~6% 横轴）；「截面/历史」切换；点击行业 → 该行业市值加权 vs 全市场市值加权历史线。

**Tab 3 · 基金分域持仓**：
- 说明「维度内多选=或，维度间=且；标签为报告期时点 PIT」。
- **12 维度多选标签组**（同 segment 12 维）+「转股期：仅转股期内」+「起算日期（默认 2017-03-31）」+「指标：市值加权% / 剔除转债ETF / 全市场对照」切换。
- **Chart_FundHoldTrend**：时间线（%）。
- **Table_FundHoldPeriods**：列 = 报告期 | 只数 | 市值加权% | 等权均值% | 中位数% | 持有市值亿 | 全市场市值加权%（2017-03 至最新 39 期）。

**数据源**：
- `GET /data/chips_index.json` → `{updated, bonds:[1009]{code,name,stk,ind,listed}}`（含已退市）。
- `GET /data/chips_market.json` → `{updated, market:[[date, factor_eq, factor_mv, ?, ?, profit_ratio], …2885], industries:[29]{name, factor, profit, n, mv}, industryDaily:{<行业>:[[date, factor_mv], …]}}`。
- `GET /data/chips/{code}.json` → `{code, name, daily:[[date, close, weighted_price, factor, profit_ratio], …1255], dist:{"<date>": [[price_bin, pct], …]}}`（三角分布拟合）。
- `GET /data/chips/fund_hold.json` → `{updated, periods:[95], codes:[1079], names:[1079], dims:{12维度→组名数组}, dimOrder:[12], rows:[13388]{p, c, w(市值加权%), ew, med, mv(亿), mkt, n, …}}`（紧凑行：period_idx, code_idx, …）。

#### 2.1.9 页面：转债个券看板（URL: `/cb/board`）

**功能定位**：存续与全量个券的基本指标与条款信息全量表。

**组件**：
- **KpiRow**：筛选结果 N 只 / 其中存续 N / 其中已摘牌 N / 存续余额合计 N 亿。
- **状态切换**：存续 / 已摘牌 / 全部（按钮组）。
- **搜索框**：placeholder「搜索代码 / 简称 / 正股…」。
- **列设置 (26/48)** dropdown + 重置 + **添加筛选条件**（动态 filter builder：字段 select + 操作符 gt/lt/contains + 值）。
- **Table_Board**：默认 26 列，全部 48 列可从列定义清单（57 key）勾选。默认列：转债代码(链接) | 转债简称 | 状态 | 类型(CB/EB) | 正股代码 | 正股简称 | 交易所 | 发行总额(亿) | 债券余额(亿) | 最新转股价 | 初始转股价 | 强赎触发价 | 下修触发价 | 回售触发价 | 票面利率(%) | 补偿利率(%) | 发行期限(年) | 面值 | 发行价格 | 到期赎回价 | 年付息次数 | 发行评级 | 最新评级 | 上市日期 | 摘牌日期 | 到期日期。
- 可选列（key 清单）：call_ratio 强赎比例(%) / reset_ratio 下修比例(%) / put_ratio 回售比例(%) / has_call 有赎回条款 / has_put 有回售条款 / call_start,call_end 强赎起止日 / put_start,put_end 回售起止日 / call_days 强赎计数(天) / call_max_days 强赎区间(天) / put_days 回售计数(天) / put_max_days / reset_days 下修计数(天) / reset_max_days / rate_type 利率类型 / value_date 起息日期 / conv_start_date 转股起始日 / conv_end_date 转股截止日 / guarantor 担保人 / guarantee_type 担保方式 / rating_comp 评级机构 / rate_clause 利率说明 / call_clause 赎回条款 / put_clause 回售条款 / reset_clause 下修条款 / conv_clause 转股条款。
- 分页 50/页；列排序；空值显示「—」。

**数据源**：`/data` 无独立文件——**静态基础信息数据集在构建期内联进 JS chunk**（`CbBoard-*.js`，dataset_name=「转债静态基础信息」，1095 行，last_updated 2026-10-29 22:41，log 含例行更新记录）。记录原始字段（Tushare 风格命名）：`ts_code, bond_full_name, bond_short_name, cb_code, cb_type, stk_code, stk_short_name, value_date(YYYYMMDD), maturity_date, rate_type, list_date, delist_date, exchange, conv_start_date, conv_end_date, conv_stop_date, rate_clause, put_clause, call_clause, reset_clause, conv_clause, guarantor, guarantee_type, issue_rating, newest_rating, rating_comp, maturity, par, issue_price, issue_size(元), remain_size(元), coupon_rate, add_rate, maturity_call_price, pay_per_year, first_conv_price, conv_price, call_ratio, reset_ratio, put_ratio, call_trig_price, reset_trig_price, put_trig_price, call_start/end, put_start/end, call_days, call_max_days, put_days, put_max_days, reset_days, reset_max_days`。
前端派生：`status = delist_date ? 已摘牌 : 存续`；`issue_size_yi = issue_size/1e8`；`remain_size_yi = remain_size/1e8`；日期 YYYYMMDD→YYYY-MM-DD。过滤规则（bundle 逆向）：剔除 `cb_type==="EB"`、剔除代码前缀 1108/1185/1240/1459/1450/1171/1210（分离交易可转债）、剔除无 list_date 且无 value_date。

#### 2.1.10 页面：可转债量化策略（URL: `/cb/quant`）

**页头口径**：「回测口径：等权、票息复权、单边千一成本」。

**组件**：
1. **StrategyCards（策略概览）**：3 张卡，每张 = 策略名 + 风格徽章（进攻/均衡/防守）+ 换仓频率（周度/月度）+ 规则描述 + 4 KPI（券数量/年化收益/年化波动/最大回撤）+ 状态行（距前高 -9.2% · 120 天未创新高 · 起始 2018-12-28）。
   - **定价高波（进攻，周度换仓）**：「BS绝对定价(预测波动率)低估前25%, 再取预测波动率前50%的高波券」（38 券，年化 28.0%）
   - **凸性优化（均衡，月度换仓）**：「gamma×vol²/价格 + 60日动量打分, 行业中性取前1/3, 月度换仓」（21 券，15.7%）
   - **优选防御（防守，周度换仓）**：「评级AA-及以上 + 剩余期限≥0.5年, 池内价格最低前30%」（63 券，12.6%）
2. **Table_PerfByRange（区间绩效）**：行 = 近1周/近1月/近3月/今年以来/近1年/近3年/全区间（年化）；列 = 定价高波/凸性优化/优选防御/中证转债指数，每格 = 收益% + 副行（回撤 -x% · 波动 x%）。
3. **Chart_Nav（净值走势）**：4 线（3 策略+中证转债指数）；「窗口起点 2018-12-28 · 各序列以窗口内首个非空净值 = 100 · 区间收益率：…」。
4. **Section_Holdings（策略持仓）**：策略 Tab×3；双口径表：
   - 实际持仓（换仓日期 2026-10-23 · 共 N 只 · 按权重降序）：代码 | 名称 | 权重(%) | 价格 | 转股溢价率(%) | 评级 | 行业。
   - 模拟调仓（以最新交易日按策略规则重算，非正式换仓点）。

**数据源**：
- `GET /data/cb_strategy/metrics.json` → `{updated, strategies:[{策略, style, rebalance, desc, 起始日期, 近1周:{收益,最大回撤,波动率}, 近1月:{…}, 近3月, 今年以来, 近1年, 近3年, 全区间:{…}}×4（含中证转债指数）]}`。
- `GET /data/cb_strategy/nav.json` → `{updated, names:[4], series:[[date, nav1, nav2, nav3, nav4]×1897]}`。
- `GET /data/cb_strategy/holdings.json` → `{updated, holdings:{定价高波/凸性优化/优选防御:{actual:{换仓日期, 持仓:[{代码,名称,权重,价格,转股溢价率,评级,行业}]}, simulated:{…}}}}`。

#### 2.1.11 页面：个券详情（URL: `/cb/bond/:code`，如 110081.SH）

**组件清单**：
1. **Header**：名称+代码、状态徽章（存续）、最新评级徽章（BBB-）、正股（*ST闻泰 600745.SH → 基本面链接）、全称（「2021年闻泰科技股份有限公司公开发行可转换公司债券 · 上交所 · 起息 2021-07-28 · 上市 2021-08-20 · 到期 2027-07-28」）。
2. **KpiGrid（8 卡）**：最新收盘价（+日期）/ 转股溢价率% / 纯债溢价率% / 到期收益率% / 纯债价值 / 转股价值 / 剩余规模(亿) / 剩余期限(年)。
3. **Chart_Candle（价格走势 OHLC + 成交量）**。
4. **Chart_Premium（溢价率 %）**：转股溢价率 + 纯债溢价率双线。
5. **Chart_ValueCompare（转股价值 vs 纯债价值 元）**：收盘价/转股价值/纯债价值 3 线。
6. **Chart_RemainSize（剩余规模与规模变动，亿元，N 次变动）** + **Table_SizeChange**：日期 | 变动原因（发行/转股/回售）| 剩余规模（亿）（逐季）。
7. **Table_RatingHistory（评级变动 N 次）**：评级日期 | 评级 | 变动（调低/维持/首次）| 前次评级。
8. **Table_IssueTerms（发行与条款信息）**：发行总额/发行期限/票面利率/补偿利率/初始转股价/最新转股价/到期赎回价/转股期/发行评级/最新评级/评级机构/担保。
9. **Accordion_Terms（条款全文×4）**：利率说明 / 赎回条款 / 回售条款 / 下修条款 / 转股条款（原始募集说明书文本）。

**数据源**：`GET /data/bonds/{code}.json`
```jsonc
{
  "code": "110081.SH",
  "basic": { ts_code, bond_full_name, bond_short_name, cb_code, cb_type, stk_code, stk_short_name,
             value_date, maturity_date, rate_type, list_date, delist_date, exchange,
             conv_start_date, conv_end_date, conv_stop_date, rate_clause, put_clause, call_clause,
             reset_clause, conv_clause, guarantor, guarantee_type, issue_rating, newest_rating,
             rating_comp, maturity, par, issue_price, issue_size, remain_size, coupon_rate, add_rate,
             maturity_call_price, pay_per_year, first_conv_price, conv_price, … },
  "daily": [[ "d":"2021-07-28", "strbv":89.96(纯债价值), "strbp":11.16(纯债溢价率%), "convv":95.43(转股价值),
              "convpr":4.79(转股溢价率%), "iv":null(隐波%), "convp":96.67(收盘价), "rt":"AA+"(当日评级),
              "osz":860000(剩余规模元), "dil":7.15(稀释率%) ] × 1272 ],
  "events": { "credit":[评级事件], "shares":[规模变动事件] }
}
```

---

### 2.2 模块：可交债（`/eb`）

#### 2.2.1 /eb/board 综合看板
- **筛选**：状态（全部/存续/已摘牌）、发行方式（公/私募）、评级。摘要「共 485 只（存续 59 / 已摘牌 426）」。
- **Table_EBBoard**：代码 | 简称 | 正股 | 公募/私募 | 状态 | 收盘价(元) | 换股价值 | 转股价 | 纯债YTM% | 剩余期限(年) | 发行规模(亿) | 存续余额(亿) | 评级 | 发行方式 | 上市板（主板/科创板/创业板）| 上市日期。无行情券（私募非交易所披露）收盘价显示「—」。
- **数据源**：`GET /data/eb/eb_board.json` → `{updated, px_date, rows:[485]{code, name, stk, stk_name, issue_type, mkt(公募/私募), board, list_date, value_date, delist_date, maturity, term_y, scale(亿), outstanding(亿), close, close_date, conv_price, convv, ytm, ptm, rating, rating_chg, live}}`。

#### 2.2.2 /eb/market 市场跟踪（Tab 私募/公募）
- **Section_MarketTotal（市场总量）**：双图——存续只数 + 当日有成交只数（左轴 0~160）；存续面值(亿) 左轴 + 成交额(亿) 右轴。
- **Section_EBIndex（EB 指数）**：「（中证估值定价，月度换仓，基点 1000）」6 线：私募EB等权 / 私募EB市值加权 / 正股等权 / 正股市值加权 / 私募EB等权(成交价) / 私募EB市值加权(成交价)。附成分只数(左)+持仓市值(亿,右)图 + **Table_EBIdxPerf**（指数 | 区间累计 | 年化 | 年化波动 | 最大回撤）。
  - 口径（原文）：「私募EB（已上市、余额>3000万、当日有中证估值）月度换仓，中证估值定价并做票息复权；虚线为同一持仓按交易所收盘价（无成交沿用前一日）定价的对照序列；正股等权/市值加权：同期持仓 1:1 对应的正股组合（同一正股多只EB占多份），价格取后复权收盘；净值在所选区间内按起点重新基点化 1000，指标按所选区间计算（年化按 243 个交易日）。」
- **Section_Trade（个券成交）**：近 20 日成交额 TOP20 横向条形；个券行情走势选择器（收盘价(元) 左轴 + 单日成交额(亿) 右轴）。
- **Section_Structure（存量结构）**：4 张分布图——评级分布 / 收盘价带分布（90-100/100-110/110-120/无成交）/ 发行规模分布（2-5/5-10/10-20/≥20亿）/ 发行方式分布；每图两套数值（亿面值 + 只数）。
- **Section_Events（条款与生命周期事件）**：类型过滤（全部465/强赎31/下修68/换股85/到期93/提前兑付10/发行175）；**Table_EBEvents**：日期 | 代码 | 简称 | 类型 | 事件（如「下修生效，转股价 72.63 元」「赎回（redem）」「换股（conv）」「到期（mature）」「发行（issue）」「公告不赎回」）；展示前 100 条 · 按日期倒序，含未来已排定事件。
- **数据源**：`GET /data/eb/eb_market.json`、`GET /data/eb_idx.json`、`GET /data/eb/detail/{code}.json`（个券行情）。

#### 2.2.3 /eb/notes 研究笔记
五章十五节研究长文（法定结构/条款体系/监管沿革/发行人与投资者结构/收益与策略/27 只公募EB全景表）。**仅本地环境提供**，生产环境显示占位。

---

### 2.3 模块：优先股（`/ps`）

#### 2.3.1 /ps/board 综合看板
- **筛选**：全部 / 银行 / 其他上市 / 非上市板；存续切换。摘要「共 29 只（存续 29 / 已退市 0）」。
- **Table_PSBoard**：代码 | 名称 | 板块 | 公司名称 | 上市日 | 规模(亿) | 股息率(%，如「3.48浮动」) | 中证估值 | 收益率(%) | 修正久期 | 收盘价(元) | 最近派息（金额+日期合并列）| 评级 | 条款徽章组（转股/强转/回售/累积/赎回）。
- 注（原文）：「收盘价仅沪深交易所（.SH/.SZ）优先股有行情，.NQ 非上市板无交易所行情，显示为中证估值。」
- **数据源**：`GET /data/ps/ps_board.json` → `{updated, px_date, rows:[103 全量]{code, name, seg(bank/listed/nq), company, live, list_date, delist_date, scale, div_rate, rate_type(固定/浮动), close, close_date, csi_val(中证估值), ytm, dur(修正久期), csi_date, last_div, last_div_date, rating, rating_date, conv, force_conv, put, cum, redeem}}`。

#### 2.3.2 /ps/market 市场跟踪
- **Section_Snapshot（市场快照）**：全部/银行/其他上市/非上市板 4 组 KPI（存续只数/规模亿/收益率中位数%/加权股息率%/久期中位数年）。
- **Chart_StockTrend（存量趋势）**：月末存续规模（亿，分板块堆叠）+ 月末存续只数（只）。
- **Chart_YearlySupply（年度供给）**：年度新发(正)/退市(负)规模（亿）+ 只数，分板块。
- **Chart_YtmTrend（收益率走势）**：板块收益率与修正久期（周度，存续券中证估值中位数）；附口径注（中证按最可能行权情景估值，行权预期变化会单日跳变；非上市板高收益率多为临近回售折价年化）。
- **Chart_DivStruct（股息率结构）**：发行股息率 1-2~8-9 分布（只数，按板块堆叠）+ 股息率类型构成（固定/浮动）。
- **Table_TermStruct（条款结构）**：条款 ×（全部/银行/其他上市/非上市板）占比% 表——可转换为普通股 58.6 / 强制转股 62.1 / 设回售 34.5 / 股息累积 37.9 / 有强制分红条件 3.4 / 有表决权 10.3 / 可赎回 100.0。
- **Chart_Dividend（派息跟踪）**：年度派息总额（亿，左）+ 派息次数（右）。
- **Table_PSEvents（事件流）**：近 24 个月，类型（派息/新发/退市）；说明如「每10股派 41.2 元（税前）」「退市（按最后数据日推断）」。
- **数据源**：`GET /data/ps/ps_market.json`。

---

### 2.4 模块：基金（`/fund`）

主页按「固收+基金 / 纯债基金 / 市场观察」三组展示 10 个模块卡 + 2 个「筹备中」。

#### 2.4.1 /fund/hf 固收+基金高频跟踪看板
- **KpiRow**：基金总数 2141 / 总规模 37,837 亿 / 一级债基 431 / 二级债基 793 / 混合基金 868 / 转债基金 49。
- **多维筛选面板**（组间取交集、组内档位取并集）：
  - 分类（4 档）、含权状态（含权/不含权）、规模（<2 / 2~10 / 10~50 / ≥50 亿 + 自定义区间）、机构占比（<20 / 20~50 / 50~80 / ≥80% + 自定义）、久期披露（<1 / 1~2 / 2~3 / 3~5 / ≥5 年 + 自定义）、股票仓位（0 / 0~5 / 5~10 / 10~20 / ≥20% + 自定义，高频估算）、转债仓位（0 / 0~10 / 10~30 / 30~50 / ≥50% + 自定义）、业绩分位（近3月/近6月/近12月 × 收益/回撤 × 0~25 / 25~50 / 50~75 / 75~100 分位 + 绝对阈值收益≥X 回撤≤Y）、货架标签（交易型/非交易型/仓位交易型/风格交易型/价值增强/成长增强/底仓增强/转债策略/其他/未分类）、波动分档（低波/中波/高波/转债基金）。
- **Table_HFBoard**（可配列 + 列设置，默认 7 列）：代码 | 名称 | 分类 | 管理人 | 规模(亿) | 近5日涨跌 | 股票仓位(披露) | 转债仓位(披露)。分页 50/页（43 页）。
- **口径说明（原文要点）**：样本=名义固收+四类（一级/二级债基、混合型=偏债混合+灵活配置滚动两期股票≤45%）∪ 转债基金实际持仓口径当前成员，剔除转债ETF 与净值停更超 30 天产品；股票/转债仓位为高频估算（**kfac 模型：卡尔曼滤波回归 + 持仓盯市，以季报披露为锚**）；业绩窗口为自然月；P 分位为同分类内排名；估算质量 MAE ≤3pp 好 / 3~5pp 中 / >5pp 弱。

#### 2.4.2 /fund/market 固收+基金市场整体观测
- **Tab 季度持仓情况**：净资产规模（亿，分类堆叠面积）+ 样本个数（右轴线）；股票仓位（季报披露 %）与转债仓位（%），各 3 线（中位数/均值/规模加权）；**Table_QuarterStats**：季度 | 个数 | 净资产(亿) | 股票仓位%（中位数/均值/规模加权）| 转债仓位%（同）——2003-09~2026-09 全量季度。
- **Tab 固收+基金业绩分布**：多窗口（近1月~近3年）收益分布与最大回撤分布（分类型分位数表+半小提琴图）；逐季度收益分布网格。
- **Tab 市场高频跟踪**：高频估算仓位时序（n=1918 等）。
- **数据源**：`GET /data/fip_market.json`。

#### 2.4.3 /fund/shelf 固收+管理人货架分布
- Kpi：管理人家数 143 / 基金总数 1640 / 总规模 32,575 亿 / 数据日期。
- **波动分档分布** 与 **策略分类分布** 双轴柱（基金数左 / 规模亿右）。
  - 波动分档口径（原文）：「按最近4个季度实际持仓（股票+转债×0.5）仓位均值划分低波（<10%）/中波（10~20%）/高波（≥20%），转债仓位均值>40% 归为转债基金」。
  - 策略三级分类口径：「按净值对专属股/债指数回归（R²）区分交易型与非交易型；非交易型按持仓九宫格与风格指数回归细分」。
- **Table_ShelfByManager**（143 家，按规模降序）：管理人 | 基金数 | 总规模(亿) | 波动分布（低/中/高/转债 n/n/n/n）| 策略分布（价值增强 n…）| 代表基金（规模前二）。
- **数据源**：`GET /data/fund_shelf.json`。

#### 2.4.4 /fund/strategy 固收+基金策略
1. **固收+基金指数**：分组（基础指数/风格指数）+ 加权方式（等权/规模加权）切换；5 指数线（一级债基固收+/二级债基固收+/混合固收+/债基固收+/整体固收+，窗口起点 2006-12-31，首值=100）；**Table_IdxPerf**（指数 | 起点 | 最新成分数 | 年化收益 | 年化波动 | 最大回撤 | 夏普 | 卡玛）；**分年度收益表**（2008~2026）。
2. **三维能力打分增强基金优选（FOF）**：基金池（整体固收+/债基固收+/二级债基/混合固收+）× 策略（均衡/稳健/进攻/防守/稳定）× 换仓频率（月/季/半年）选择器；净值+超额净值（=策略/S0，右轴）；绩效表（年化收益/波动/回撤/夏普/卡玛/年化超额/信息比率/超额最大回撤）；分年度收益与超额；**当前持仓（双口径）**：实际持仓（换仓日期 2026-09-30 · 共 8 只 · 等权满仓）与模拟调仓两表，列 = 名称 | 代码 | 权重(%) | 综合仓位 | 得分 | 赢率 | 弹性 | 不赔比率 | 股票β CV | 债券β CV。
   - 三维能力口径（原文要点）：「进攻=上涨窗口仓位中性化赢率+弹性；防守=下跌窗口不赔比率；策略稳定性=基金日收益对中证800与1-3Y优选信用债指数(CBA20921.CS)的滚动60日beta，取年内5个时点计算变异系数 CV=std/|mean|，越小越稳；打分窗口 250 个交易日；市场涨跌按中证800 滚动21日划分；综合仓位=最近2期披露 (股票+0.5×转债)/净资产；样本规模>5亿；各维度 top8 等权满仓」。
3. **固收+画线基金组合**：画线组合 vs 货基指数(885009.WI) 净线+绩效+分年度+双口径持仓（10 只等权）。
   - 口径（原文要点）：「243 交易日窗口内月度创新高概率>0.5，且标准化净值对货基指数的调整欧氏距离（下跌段权重×2）/样本数 ≤0.025；入选超 10 只取距离最小 10 只等权，允许空仓月；月度换仓」。
- **数据源**：`/data/fip_strategy/{nav_index, index_meta, enh_nav, enh_meta, holdings}.json`。

#### 2.4.5 /fund/pb 纯债基金跟踪看板
- Kpi：基金总数 1674 / 总规模 46,515 亿 / 分类只数（中长期纯债 1182 / 短债债基 376 / 一级债基 85 / 二级债基 19 / 偏债混合 6 / 灵活配置 6）。
- 筛选：分类 + 规模档。
- **Table_PBBoard**：代码 | 名称 | 分类 | 管理人 | 基金经理 | 管理费率 | 机构占比 | 规模(亿) | 最新净值 | 日涨跌 | 今年以来 | 杠杆率 | 久期 | 久期(估算) | 净值日期。
- 口径：「中长期纯债/短债及类纯债基金（最新季报无股票/转债持仓，剔除定开与非日度披露）」。
- **数据源**：`GET /data/pb_index.json`。

#### 2.4.6 /fund/pb-duration 纯债基金久期跟踪
- **Chart_GroupDuration（分组久期走势）**：分类（中长期纯债/短债债基）× 券种类型（利率/信用/类货币/混合）× 指标（中位久期 / 分歧度(75-25分位) / 变异系数）切换；实线=Kalman 日度估算中位（ewm5 平滑，仅展示层）· 圆点=半年度披露中位。
- **Table_LatestRead（最新读数）**：分类 | 券种类型 | 样本只数 | 估算中位 | 估算均值 | 分歧度(75-25) | 变异系数 | 最近期披露中位(日期) | 差值(估算-披露)。
- **Chart_SampleSize（样本量演变）**：逐季度末样本只数（分类堆叠）。
- 口径（原文要点）：「逐季度末 PIT 确定样本（分类∈中长期纯债/短债债基，最新单季股+转债=0，剔定开与非日度披露），当季样本用于下一季度聚合；**Kalman 无约束时变系数回归（burn=60, q_beta=1e-4）**，指数集=core10换5年地债(930865)+30年国债(931080, 2023-12-19 起拼接接入)；披露久期为半年报/年报组合久期中位数。」
- **数据源**：`/data/pb_duration/{meta, group_daily, vs_disclosed}.json`。

#### 2.4.7 /fund/pb-shelf 纯债管理人货架分布
- Kpi：157 家 / 2864 只 / 84,917 亿 / 分类基准日。
- **Chart_ShelfTrend（逐季度货架结构走势）**：季度选择 + 维度（名义类型/久期档/券种类型/摊余与否）切换；规模堆积+占比堆积双图；「季度末按现行口径 PIT 重分类；末点为最新交易日」。
- 4 张分布图（名义类型 / 非摊余×券种类型 / 非摊余×久期档 / 摊余 vs 非摊余），均基金数(左)+规模(亿,右)双轴。
- **Table_PBShelf**（157 家）：管理人 | 基金数 | 总规模 | 名义分布 | 久期分布（非摊余）| 券种类型 | 代表基金（规模前二）。
- 口径要点：券种类型=滚动2季券种占比均值>60% 分类（优先级 利率→信用→类货币，其余混合）；久期档=最近已生效半年点利率敏感度久期 <1/1-3/3-5/≥5 年；摊余成本法按 Wind 估值方法识别；指数债基仅场外份额。
- **数据源**：`GET /data/pb_strategy/shelf.json`。

#### 2.4.8 /fund/lowvol-strategy 纯债基金指数
- **Tab 纯债指数**：分组（基础指数/券种风格/杠杆/久期子类）+ 加权（等权/规模加权）；短债债基/中长期纯债基 2 线（起点 2013-03-31）；绩效表+分年度表。
- **Tab 纯债增强指数**：核心/卫星/分层优选策略。
- 口径要点：样本=分类代码 2001010301/2001010302；剔定开与最短持有>3个月份额；纯度=最近≤4期季报股+转债均0；券种风格阈值 60%；杠杆指数要求滚动2季杠杆率>1.25；久期子类按披露利率敏感度久期近4个半年点（min≥4 长久期/max≤3 中短/极差>2 轮动/其余中等）；规模门槛短债≥2亿、中长期≥5亿；季度末换仓、公告日 PIT。
- **数据源**：`/data/pb_strategy/{nav_index, index_meta, strategy_nav, strategy_meta, holdings}.json`。

#### 2.4.9 /fund/bond-market 债券型基金市场综合看板
- 页头口径（原文）：「规模分存续/新发（新发未披露按发行规模估算）；规模变动分解：Δ规模=净值贡献+申赎+新发+移入−退出，净值贡献按复权收益（分红视同流出计入申赎），移入=非新发的口径升降/数据补缺；持仓类指标仅披露组；杠杆率=总资产/净资产；分券种/杠杆的固收+组别为其纯债底仓口径；全市场基准：股票=全A总市值加权(2015年起)、转债=全部存续转债市值加权」。
- **AnchorNav 9 节**：规模与只数 / 纯债产品形式 / 申赎与资金流 / 股票仓位 / 持股风格 / 转债仓位 / 转债持仓风格 / 分券种配置 / 杠杆率。
- 组件模式：每节 = 堆积面积(或柱)图（阶段/分类筛选、堆积面积/折线切换）+ 明细表。例「名义分类规模」：八分类（一级债基/二级债基/混合型基金/转债基金/中长期纯债/短债债基/指数债基-ETF/指数债基-非ETF）× 存续/新发；明细表列出每报告期 16 数值列+合计。
- 「下载规模明细 CSV」。
- **数据源**：`GET /data/bond_fund_board.json`。

#### 2.4.10 /fund/pension 养老金产品跟踪
- **一级类型等权指数**：债券型/混合型/股票型/货币市场型 4 线（基点 1000，2013-08 起）+ 绩效表（类型 | 产品数 | 起基日 | 最新值 | 年化收益 | 年化波动 | 最大回撤）。
- **二级类型等权指数**：12 个二级类型（固定收益型-普通/存款/债权计划/商业银行理财产品/信托产品/债券、优先股型、股票型/股票专项型/股权型、混合型、货币型）归属一级；4 组 Tab。
- **固收-普通产品 RBSA 仓位跟踪**：仓位结构堆叠（二级债基/纯债合计/股票合计/转债合计/存款非标合计/现金，组内等权均值%，周度，现金为负=杠杆）；久期跟踪（Σ纯债权重×指数修正久期，p25~p75 区间+中位+均值）；纯债仓位细分（信用债/国开债/超长债/高收益债）；股票仓位细分（800价值/周期/小盘/成长/消费/科技/红利低波）；其他资产细分。
  - 样本池口径（原文）：「纯债=中证信用债/高收益债/国开债/超长债；股票=中证800全收益+成长/价值/红利低波/周期/科技/消费/小盘；转债=市值加权/等权（等权经中性化）；其他=二级债基(885007.WI 经 800+信用债中性化)/存款/非标（上限10%）；现金=货币基金指数；股票合计受 ≤15% 约束」。
- **RBSA 最新截面表**（可排序）：代码 | 名称 | 管理人 | 纯债 | 转债 | 存款+非标 | 二级债基 | 股票 | 权益中枢 | 现金 | 久期(年) | R²；下载 CSV。
- **规模与加权仓位 / 分管理人敞口** 节。
- **数据源**：`/data/pension/{idx, idx_meta, rbsa, manager, scale}.json`。

---

### 2.5 模块：ETF 策略（`/etf`）

#### 2.5.1 /etf/board 个券看板
- Kpi：在场 1698 只 / 总规模 50,414 亿 / 当日成交额 5,697.8 亿 / 今年收益中位数 -10.47%。
- 筛选：分类（全部/股票/QDII/商品/债券/货币）+ 列设置 (12/17) + 筛选条件构建器。
- **Table_ETFBoard** 默认 12 列：代码 | 简称 | 分类 | 收盘价 | 规模(亿) | 成交额(亿) | 折溢价% | 1周% | 1月% | 今年% | 近3年% | 今年最大回撤%。
- **数据源**：`GET /data/etf_index.json`。

#### 2.5.2 /etf/scale 市场结构看板
- Kpi：总规模/总只数/近1月规模变动/年内新上市。
- 图：各类 ETF 规模堆叠面积（QDII/债券/商品/股票/货币，近3年）；只数；**月度规模变动拆解**（份额贡献 vs 净值贡献，最近36月）；月度新上市只数（五分类堆叠）。
- **数据源**：`GET /data/etf_scale.json`。

#### 2.5.3 /etf/pb 纯债 ETF 综合看板
- 口径：纯债ETF（剔除转债/可交换债ETF）；大类=信用债/利率债；品种按简称与跟踪指数关键词划分（做市公司债/公司债/国债/地方债/城投/政金/短融/科创债）。
- Kpi：在场 51 只 / 总规模 10,370.1 亿 / 只均 YTD +2.89% / YTD 份额申赎净流入 +2,502.8 亿。
- 图：规模走势（按品种堆叠，近36月）；月度规模变动拆解（份额/净值贡献）；**品种汇总表**（品种 | 只数 | 规模亿 | 占比% | 只均YTD% | YTD申赎亿）。
- 明细筛选：大类/规模档/品种/跟踪指数（24 个指数全清单）。
- **Table_PBETF**：代码 | 简称 | 大类 | 品种 | 跟踪指数 | 管理人 | 收盘价 | 规模(亿) | 份额(亿份) | 成交额(亿) | 折溢价% | 1周% | 1月% | 3月% | 今年% | 近3年% | 今年最大回撤% | 近3年最大回撤% | 久期 | 久期披露日 | 上市日期。
- **数据源**：`GET /data/pb_etf_index.json`、`GET /data/pb_etf_market.json`。

#### 2.5.4 /etf/stock-etf 股票 ETF 策略
- 页头口径：「月度换仓，等权、复权价、零成本」。
- 3 张策略卡（结构同 /cb/quant）：
  - **相对动量（进攻，月度）**：「120日动量前25%为候选, 按动量降序、两两相关性<0.9依次入选」（11 券，年化 7.9%）
  - **资金流补充动量（均衡，月度）**：「大单+特大单净买入前25%且买入占比未过热(≤75%分位), 叠加120日动量前25%」（4 券，17.1%）
  - **波动率控制动量（防守，月度）**：「120日动量前25%, 且短期波动率/均值相对长期抬升(短长波动差>0)」（0 券=当前空仓，16.6%）
  - 对照：股票ETF等权。
- 区间绩效表 / 净值走势 / 双口径持仓（列：代码 | 名称 | 权重% | 价格 | 规模(亿) | 跟踪指数）。
- **数据源**：`/data/stk_etf_strategy/{metrics, nav, holdings}.json`。

---

### 2.6 模块：固收+策略（`/strategy`）

#### 2.6.1 /strategy/income-equity 生息固收+权益策略 · 红利增强
- 页头口径：「红利增强选股策略的回测表现、与中证红利全收益（H00922.CSI）的对比及最新择券 · 月度换仓、双边千一成本」。
- 策略卡：**红利增强（防守，月度换仓）**：「中证全指成分内, ROE稳定性过滤+红利质量评分Top50+动量波动精选, 逆波动加权+申万一级行业10%压顶, 双边千一成本」（40 券，年化 17.0%，年化超额 +7.6%）。
- 区间绩效（红利增强 vs 中证红利全收益 + 超额列）；**分年度收益表**（2010~2026 + 超额列）；净值走势（2 线）；**双口径持仓**：实际持仓（换仓日期 2026-09-30 · 40 只 · 按权重降序 · 附选股漏斗「全指成分 5035 只 → ROE稳定性过滤 4469 只 → 红利评分Top50 50 只 → 动量波动过滤后 40 只」），列 = 代码 | 名称 | 申万一级 | 权重(%) | 股息率(%)；注「股息率为调仓日价格/股息口径的倒数（数据源日频前填充值）」。
- **数据源**：`/data/div_strategy/{metrics, nav, holdings}.json`。

#### 2.6.2 /strategy/multi-asset 多资产配置模型 · 股债两资产实测
- 页头口径：「股：中证800全收益 (H00906.CSI) · 债：中债-商业银行二级资本债券AAA财富 (CBA26301.CS) · 月度调仓（每月最后交易日），零交易成本，收盘价成交 · 指标自 2017 年起（基期 2016 年末 = 100）」。
- **Table_StrategyOverview（指标总览，44+ 行可排序可筛选）**：
  - 筛选：指标起点（2017~2025）；类别（固定比例/风险驱动/收益驱动/保险策略）；指标阈值（年化收益≥/夏普≥/卡玛≥/回撤≤）。
  - 列：类别 | 策略 | 参数 | 年化收益 | 年化波动 | 最大回撤 | 夏普 | 卡玛 | 平均仓位 | 方向准确率 | 加仓命中 | 减仓命中 | 择时增益/月。
  - 策略族：恒定比例（股5/债95 ~ 股30/债70）；风险预算（股10%~70%）、风险平价、目标波动（2%~5%）、CVaR 尾部风险优化（90/95/99%）；约束均值方差 CMVO（λ=10/100/1000）、动量波动协同 MVS（10%~50%）；CPPI（m=0.5/1/2/3）、TIPP（m=0.5/1/2/3）、OBPI 期权保护（3%~10%）；+ 两行资产腿（中证800全收益 / 二级资本债）。
  - 指标定义（原文）：「最大回撤为幅度 · 平均仓位为期间月末股权重均值 · 命中率=调仓方向与权益次月涨跌一致的占比（恒定比例不调仓、资产腿无此项）· 择时增益/月 = Σ sign(Δw)×次月收益 ÷ 调仓次数」。
- **Section_StrategyDetail（策略详情）**：类别 Tab（固定比例/风险驱动/收益驱动/保险策略）→ 策略子 Tab → 参数选择（如权益中枢 k：股5/债95…）；每策略：说明卡（规则文字+公式）+ 5 KPI + **组合净值 vs 债券净值（可加自选参照 combobox——其余 9 策略/中证800）** + 仓位变化（月度股权重，末点=模拟调仓）+ 回撤图 + 分年度表现表（年度 | 收益 | 年化波动 | 最大回撤 | 平均仓位；含「迄今为止累计」「累计年化」合计行）。
- **数据源**：`GET /data/alloc_strategy/strategies.json`。

#### 2.6.3 /about 关于本站
- 网站更新日志（倒序，日期 + 若干条更新说明；2026-10-20 ~ 至今）。该日志是各功能口径的第一手说明。

---

## 3. 底层数据模型与 Schema 逆向设计

### 3.1 数据源分类（现实中来源推断）

| 数据类别 | 现实来源 |
|---|---|
| 转债/EB/正股行情与基础条款 | Tushare / Wind / 交易所披露（字段命名 `ts_code/conv_price/...` 为 Tushare `cb_basic` 风格；评级、募集条款文本来自募集说明书） |
| 指数行情 | 中证指数（中证转债 000832、中证800全收益 H00906、中证红利全收益 H00922）、中债（CBA26301 二级资本债财富、CBA20921 信用债） |
| 交易所投资者结构 | 沪深交易所月度可转债投资者持有结构披露 |
| ETF 份额/规模 | 交易所 ETF 份额披露（变动日记录）+ 基金公司 |
| 基金季报/持仓 | 基金定期报告（股票/转债/券种配置、杠杆、久期、机构占比） |
| 基金高频净值 | 基金公司/天相/朝阳永续式日频净值库 |
| 养老金产品净值 | 人社部养老金产品披露 |
| 中证估值 | 中证指数估值服务（EB/优先股估值） |
| 衍生计算（站内自研） | 百元溢价率回归、退市调整 BS 希腊字母、五因子归因、筹码三角分布、kfac 高频仓位、Kalman 久期、RBSA、各策略回测 |

### 3.2 静态数据文件全集（前端契约层）

```
/data/daily_brief.json                     首页晨报
/data/cb_market.json                       市场日度 28 列面板 + 持有人矩阵
/data/cb_idx.json                          5 指数日序列
/data/cb_supply.json                       供给月度/年度
/data/cb_greeks.json                       希腊字母 4 块
/data/cb_etf.json                          转债ETF 时序
/data/cb_reports/index.json|{YYYY-MM}.json|explain.json   月度复盘
/data/cb_segment_idx.json|cb_segment_idx_style.json|cb_segment_hold.json  分域
/data/clause_board.json|clause_events.json 条款看板与事件
/data/chips_index.json|chips_market.json|chips/{code}.json|chips/fund_hold.json  筹码
/data/bonds/{code}.json                    个券详情
/data/bonds_index.json                     全部转债历史净值索引(name,rows,start,end,bh)
(转债静态基础信息 1095 行内联于 CbBoard JS chunk)
/data/eb/eb_board.json|eb_market.json|eb_idx.json|eb/detail/{code}.json   EB
/data/ps/ps_board.json|ps_market.json      优先股
/data/funds_index.json                     固收+基金索引
/data/fund_shelf.json                      固收+货架
/data/fip_market.json                      固收+市场季度
/data/fip_strategy/{nav_index,index_meta,enh_nav,enh_meta,holdings}.json
/data/pb_index.json                        纯债看板
/data/pb_duration/{meta,group_daily,vs_disclosed}.json
/data/pb_strategy/{shelf,nav_index,index_meta,strategy_nav,strategy_meta,holdings}.json
/data/bond_fund_board.json                 债基市场
/data/pension/{idx,idx_meta,rbsa,manager,scale}.json
/data/etf_index.json|etf_scale.json        ETF
/data/pb_etf_index.json|pb_etf_market.json 纯债ETF
/data/stk_etf_strategy/{metrics,nav,holdings}.json
/data/cb_strategy/{metrics,nav,holdings}.json
/data/div_strategy/{metrics,nav,holdings}.json
/data/alloc_strategy/strategies.json
```

### 3.3 核心 TypeScript Schema（按上文各节字段归纳）

```ts
// ============ 维度表 ============
interface BondDim {            // 转债/EB 静态信息（48 列看板数据集）
  ts_code: string;             // "110081.SH"
  bond_full_name: string; bond_short_name: string;
  cb_type: "CB" | "EB";
  stk_code: string; stk_short_name: string;
  exchange: "SH" | "SZ";
  value_date: string; maturity_date: string; list_date: string; delist_date: string | null; // YYYYMMDD
  conv_start_date: string; conv_end_date: string; conv_stop_date: string | null;
  issue_size: number;          // 元
  remain_size: number;         // 元
  par: number; issue_price: number; maturity: number;   // 面值/发行价/期限(年)
  coupon_rate: number; add_rate: number; maturity_call_price: number; pay_per_year: number;
  first_conv_price: number; conv_price: number;
  call_ratio: number; reset_ratio: number; put_ratio: number;       // 130/85/70 (%)
  call_trig_price: number | null; reset_trig_price: number | null; put_trig_price: number | null;
  call_start: string | null; call_end: string | null;
  put_start: string | null; put_end: string | null;
  call_days: number; call_max_days: number;      // 已计数/窗口
  put_days: number;  put_max_days: number;
  reset_days: number; reset_max_days: number;
  issue_rating: string | null; newest_rating: string | null; rating_comp: string | null;
  guarantor: string | null; guarantee_type: string | null; rate_type: string | null;
  rate_clause: string; call_clause: string; put_clause: string; reset_clause: string; conv_clause: string;
}

interface FundDim {
  code: string; name: string; manager: string;      // 管理人
  category: "一级债基" | "二级债基" | "混合基金" | "转债基金" | "中长期纯债" | "短债债基" | …;
  managers: string;          // 基金经理（顿号分隔）
  mgmt_fee: number; inst_hold_pct: number; scale: number;   // 管理费率/机构占比/规模(亿)
}

// ============ 事实表 ============
interface CbMarketDailyFact {   // cb_market.json daily.rows（28 列紧凑数组，见 2.1.2 列定义）
  date: string;                // YYYYMMDD
  n: number; bal: number; mv: number;         // 只数/余额/市值(亿)
  conv_v: [number, number, number];           // 转换价值 中位/均值/市值加权
  bond_v: [number, number, number];           // 债底价值
  ptm: [number, number, number];              // 剩余期限(年)
  ytm: [number, number, number];              // YTM(%)
  conv_prem: [number, number, number];        // 转股溢价率(%)
  bond_prem: [number, number, number];        // 债底溢价率(%)
  prem100: number; prem100_n: number;         // 百元溢价率(%)与回归样本数
  abn_n: number; abn_pct: number;             // 异常标的
  turnover: [number, number, number];         // 换手率(%) 中位/均值/市值加权
}

interface BondDailyFact {       // /data/bonds/{code}.json daily 行
  d: string;                    // YYYY-MM-DD
  strbv: number | null;         // 纯债价值(元)
  strbp: number | null;         // 纯债溢价率(%)
  convv: number | null;         // 转股价值(元)
  convpr: number | null;        // 转股溢价率(%)
  iv: number | null;            // 隐含波动率(%)
  convp: number | null;         // 收盘价(元)
  rt: string | null;            // 当日债项评级
  osz: number | null;           // 剩余规模(元)
  dil: number | null;           // 稀释率(%)
}

interface GreeksFact { d: string; median: number; mean: number; mv: number; n: number;
  iv?: number; vol_his?: number; d_term?: number; g_term?: number; t_term?: number; }

interface AttributionFact { m: string; n: number; ret_actual: number;
  delta: number; gamma: number; theta: number; vega: number; rho: number; residual: number; }

interface ClauseFact {          // clause_board call/reset/put 行
  code: string; name: string; stk_name: string;
  cb_close: number; convv: number; conv_price: number; ptm: number;
  ratio: number; window: number; need: number; cnt: number | null; gap: number | null;
  status: "triggered" | "counting" | "proposed" | "exempt" | "satisfied" | …;
  exempt_end: string | null; last_event: string | null;
  last_reset?: { date: string; price: number }; note: string | null; days_to?: number | null;
}

interface StrategyMetrics { 策略: string; style: string; rebalance: string; desc: string;
  起始日期: string; "近1周"|"近1月"|"近3月"|"今年以来"|"近1年"|"近3年"|"全区间": { 收益: number; 最大回撤: number; 波动率: number }; }

interface HoldingRow { 代码: string; 名称: string; 权重: number; 价格?: number;
  转股溢价率?: number; 评级?: string; 行业?: string; 规模?: number; 跟踪指数?: string; }
```

### 3.4 关系型 DDL（重建用，PostgreSQL 方言）

```sql
-- 维度：债券
CREATE TABLE dim_bond (
  ts_code        VARCHAR(12) PRIMARY KEY,
  bond_short_name VARCHAR(40) NOT NULL,
  cb_type        CHAR(2) NOT NULL,              -- CB / EB
  mkt            VARCHAR(4),                    -- 公募/私募(EB)
  stk_code       VARCHAR(12), stk_short_name VARCHAR(40),
  exchange       CHAR(2), board  VARCHAR(8),
  value_date DATE, maturity_date DATE, list_date DATE, delist_date DATE,
  conv_start_date DATE, conv_end_date DATE, conv_stop_date DATE,
  issue_size  NUMERIC(14,2), remain_size NUMERIC(14,2),      -- 元
  par NUMERIC(8,2), issue_price NUMERIC(8,2), maturity NUMERIC(4,1),
  coupon_rate NUMERIC(5,2), add_rate NUMERIC(5,2),
  maturity_call_price NUMERIC(6,2), pay_per_year SMALLINT,
  first_conv_price NUMERIC(10,4), conv_price NUMERIC(10,4),
  call_ratio SMALLINT, reset_ratio SMALLINT, put_ratio SMALLINT,
  issue_rating VARCHAR(8), newest_rating VARCHAR(8), rating_comp VARCHAR(60),
  guarantor VARCHAR(60), guarantee_type VARCHAR(20),
  rate_clause TEXT, call_clause TEXT, put_clause TEXT, reset_clause TEXT, conv_clause TEXT,
  updated DATE
);
CREATE INDEX idx_bond_type ON dim_bond (cb_type, delist_date);

-- 事实：债券日行情与估值
CREATE TABLE fact_bond_daily (
  ts_code VARCHAR(12), trade_date DATE,
  close NUMERIC(9,3), conv_v NUMERIC(9,3), bond_v NUMERIC(9,3),
  conv_prem NUMERIC(7,3), bond_prem NUMERIC(7,3), ytm NUMERIC(7,3),
  iv NUMERIC(7,3), delta NUMERIC(6,4), turnover NUMERIC(7,3),
  amount NUMERIC(14,3), remain_size NUMERIC(14,2), rating VARCHAR(8),
  PRIMARY KEY (ts_code, trade_date)
);
CREATE INDEX idx_bd_date ON fact_bond_daily (trade_date);

-- 事实：市场日度截面（cb_market 28 列）
CREATE TABLE fact_cb_market_daily (
  trade_date DATE PRIMARY KEY,
  n SMALLINT, bal NUMERIC(10,1), mv NUMERIC(10,1),
  conv_v_med NUMERIC(8,2), conv_v_avg NUMERIC(8,2), conv_v_mv NUMERIC(8,2),
  bond_v_med NUMERIC(8,2), bond_v_avg NUMERIC(8,2), bond_v_mv NUMERIC(8,2),
  ptm_med NUMERIC(5,2), ptm_avg NUMERIC(5,2), ptm_mv NUMERIC(5,2),
  ytm_med NUMERIC(7,2), ytm_avg NUMERIC(7,2), ytm_mv NUMERIC(7,2),
  conv_prem_med NUMERIC(7,2), conv_prem_avg NUMERIC(7,2), conv_prem_mv NUMERIC(7,2),
  bond_prem_med NUMERIC(7,2), bond_prem_avg NUMERIC(7,2), bond_prem_mv NUMERIC(7,2),
  prem100 NUMERIC(6,2), prem100_n SMALLINT,
  abn_n SMALLINT, abn_pct NUMERIC(5,2),
  turn_med NUMERIC(6,2), turn_avg NUMERIC(6,2), turn_mv NUMERIC(6,2)
);

-- 事实：希腊字母日度 / 归因月度
CREATE TABLE fact_cb_greeks_daily (
  trade_date DATE, median NUMERIC(6,3), mean NUMERIC(6,3), mv NUMERIC(6,3), n SMALLINT,
  exp_ret_med NUMERIC(7,3), exp_ret_mean NUMERIC(7,3), exp_ret_mv NUMERIC(7,3), n_exp SMALLINT,
  iv_med NUMERIC(6,2), vol_his_med NUMERIC(6,2),
  d_term NUMERIC(6,2), g_term NUMERIC(6,2), t_term NUMERIC(6,2),
  above_ytm_n SMALLINT, above_ytm_pct NUMERIC(5,1),
  PRIMARY KEY (trade_date)
);
CREATE TABLE fact_cb_attribution_monthly (
  month CHAR(7) PRIMARY KEY, n SMALLINT,
  ret_actual NUMERIC(7,3), delta NUMERIC(7,3), gamma NUMERIC(7,3),
  theta NUMERIC(7,3), vega NUMERIC(7,3), rho NUMERIC(7,3), residual NUMERIC(7,3)
);

-- 事实：条款事件
CREATE TABLE fact_clause_event (
  id BIGSERIAL PRIMARY KEY, ts_code VARCHAR(12), event_date DATE,
  event_type VARCHAR(8),          -- call 强赎 / reset 下修 / put 回售 / conv 换股 / mature 到期 / issue 发行
  executed BOOLEAN,               -- 执行/不执行（或事件子型）
  detail TEXT,                    -- "下修生效，转股价 72.63 元"
  ret_5d NUMERIC(7,3), ret_10d NUMERIC(7,3), ret_20d NUMERIC(7,3),
  excess_5d NUMERIC(7,3), excess_10d NUMERIC(7,3), excess_20d NUMERIC(7,3)
);

-- 事实：供给（月度/年度）
CREATE TABLE fact_cb_supply_monthly (
  month CHAR(7) PRIMARY KEY,
  issue_n SMALLINT, issue_sz NUMERIC(9,1), conv_n SMALLINT, conv_sz NUMERIC(9,1),
  other_n SMALLINT, other_sz NUMERIC(9,1), dig_conv NUMERIC(9,1), dig_cash NUMERIC(9,1)
);

-- 事实：基金季报持仓 / 高频仓位 / 久期（略列，同 3.3）
CREATE TABLE fact_fund_hf_position (
  fund_code VARCHAR(12), trade_date DATE,
  stock_pos NUMERIC(6,2), cb_pos NUMERIC(6,2),       -- kfac 估算(%)
  stock_pos_disc NUMERIC(6,2), cb_pos_disc NUMERIC(6,2),  -- 季报披露
  PRIMARY KEY (fund_code, trade_date)
);
CREATE TABLE fact_fund_duration (
  fund_code VARCHAR(12), trade_date DATE, dur_kalman NUMERIC(5,2),
  group_key VARCHAR(24),           -- 中长期纯债/短债 × 利率/信用/类货币/混合
  PRIMARY KEY (fund_code, trade_date)
);
```

---

## 4. 前后端 API 契约规范（重建用）

### 4.1 动态 API（仅鉴权）

**POST /api/auth/login**
```json
// Request
{ "username": "evian", "password": "••••" }
// Response 200
{ "ok": true, "user": { "username": "evian", "displayName": "EvianZhou" } }
// Response 401
{ "ok": false, "error": "用户名或密码错误" }
```

**GET /api/auth/me**（携带 Cookie）
```json
// 200
{ "authenticated": true, "username": "evian", "displayName": "EvianZhou" }
// 401 → 前端重定向登录页
```

**POST /api/auth/register**（注册申请，管理员审核）
```
{ "name": "...", "username": "...", "email": "公司邮箱", "password": "≥8位且含大写字母与数字" }
```

### 4.2 静态数据端点（核心组件 Response 样例）

**GET /data/daily_brief.json** → 见 2.0。
**GET /data/cb_market.json**
```json
{ "updated": "2026-10-29",
  "daily": { "cal": ["20041019", "...", "20261029"],
             "rows": [[319, 4737.7, 5967.95, 81.32, 91.43, 94.79, 106.55, 105.57, 106.6,
                        2.69, 3.0, 2.59, -3.53, -9.65, -9.06, 59.34, 73.55, 63.56,
                        19.83, 36.37, 30.46, 41.78, 293, 6, 1.88, 4.21, 16.14, 8.64]] },
  "holder": { "sse": { "months": {"0": "2016-07"}, "investors": {"0": "公募基金"},
                       "hold": {"0": {"0": 125.74}}, "trade": {"0": {"0": 67.33}},
                       "holdUnit": "面值(亿元)", "tradeUnit": "成交金额(亿元)" },
              "szse": { … } } }
```
**GET /data/cb_greeks.json / cb_etf.json / cb_segment_hold.json / clause_board.json / eb/eb_board.json / ps/ps_board.json / cb_strategy/*.json** → 见 2.1~2.6 各节样例。

### 4.3 前端渲染契约要点（重建时必须遵守）

1. **紧凑数组**：所有时序优先用「列数组/行数组」而非对象数组（cb_market.rows、etf.series、nav.series 均为 `[date, v1..vk]` 行数组），以控制体积（cb_market.json 449KB / cb_idx 208KB / 看板静态信息内联 chunk 757KB）。
2. **窗口重算在前端**：切换 ChartRangePicker 后净值重基点化与绩效指标（累计/年化/波动/回撤/夏普/卡玛）均前端即时计算，无后端参与；年化按 243 交易日。
3. **双口径持仓**：所有策略页持仓区固定两表——`实际持仓`（最近正式换仓日）与`模拟调仓`（最新交易日重算），标题行注明日期与只数。
4. **图例可点击隐藏**（ECharts 默认）；双轴图右轴常为只数/累计/占比。
5. **表格规范**：可排序列头；空值「—」；代码列内链到详情页；分页默认 50/页。
6. **下载**：每个图表 CSV 导出按当前时间窗切片。
7. **月度复盘/研究笔记**：index.json 驱动的 combobox + 懒加载 `{YYYY-MM}.json`（title + body 长文，body 为 Markdown/HTML 富文本）。

---

## 附录 A：站方口径原文索引（重要公式速查）

| 指标 | 公式/口径 |
|---|---|
| 百元溢价率 | 每日截面回归 y = a·e^(-b·x) + c（y=转股溢价率，x=转换价值/100），取 x=1 拟合值；样本=未公告强赎且剩余期限>0.5年 |
| 换手率 | 每日成交金额 / 总市值；市值加权以个券总市值为权重 |
| 异常标的 | 价格≥130 且 转股溢价率≥30% 且 换手率≥100% |
| 转债理论价 | V = 存活概率 × [ s·N(d1) + X·e^(-r·t)·(1−N(d2)) ]，r=债底YTM，退市线=max(1/股价, 3亿/总市值)×平价 |
| Delta | ∂V/∂s = N(d1)·存活概率 + 退市项修正 |
| 预期收益率(年化) | ( Δ·s·r_f + ½·Γ·(s·σ_his)² + Θ ) / P，r_f=4% |
| 高于债底占比 | #(E[ret] > 债底YTM) / 有效样本只数 |
| 月度归因 | 月收益 = Δ项+Γ项+Θ项+vega项+ρ项+残差（全重定价：依次替换 s→σ→t→r） |
| ETF 规模 | 总份额 × 收盘价；份额变动日记录、交易日间前向填充 |
| 净申购估算 | 当日总份额变动 × 收盘价 |
| 指数（转债/EB） | 月度换仓基点1000；票息复权；年化按 243 交易日；余额>3000万剔除触发强赎/摘牌 |
| 波动分档（固收+） | 最近4季 (股票+转债×0.5) 均值：<10% 低波 / 10~20% 中波 / ≥20% 高波；转债均值>40% 归转债基金 |
| kfac 高频仓位 | 卡尔曼滤波回归 + 持仓盯市，以季报披露为锚；MAE ≤3pp 好 / 3~5pp 中 / >5pp 弱 |
| Kalman 久期 | 无约束时变系数回归（burn=60, q_beta=1e-4）；指数=core10换5年地债(930865)+30年国债(931080) |
| 券种类型分类 | 滚动2季券种占比均值>60%（优先级 利率→信用→类货币，其余混合） |
| 综合仓位（FOF 打分） | 最近2期披露 (股票 + 0.5×转债) / 净资产 |
| 择时增益/月（多资产） | Σ sign(Δw)×次月收益 ÷ 调仓次数 |
| 三维能力 | 进攻=上涨窗口仓位中性化赢率+弹性；防守=下跌窗口不赔比率；稳定=股债两腿滚动60日beta 的 CV rank 之和 |
| 股息率（红利增强） | 调仓日价格/股息口径的倒数（日频前填充） |

## 附录 B：重建验收清单

- [ ] 7 大顶级模块 + 27 个子页面路由全部可导航（含返回链接与二级 Tab）
- [ ] 登录墙：未登录任意路由 → `/` 登录页（登录/注册 Tab）
- [ ] 每图表具备：时间窗按钮组（全部/近5年/近3年/近1年/今年以来/自定义起点）+ CSV 下载
- [ ] 双口径持仓表（实际/模拟）出现在全部 5 个策略页
- [ ] 指数绩效表四件套：累计/年化/年化波动/最大回撤
- [ ] 暗/浅主题切换且图表色板联动
- [ ] 全部 /data/*.json 按第 3.2 节清单产出，`updated` 字段与页面「数据截至」一致
