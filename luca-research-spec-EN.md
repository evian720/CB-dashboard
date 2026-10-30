# Luca Research (luca-research.com) — Technical & Data Specification (English Edition)

> Purpose: This document was produced by fully reverse-engineering https://luca-research.com/ and can be fed as a Prompt to an LLM to rebuild the site's data layer and frontend/backend logic 1:1.
> Site theme: a research dashboard for China A-share convertible bonds (CB), exchangeable bonds (EB), preferred stocks, fixed-income-plus ("固收+") funds, ETFs, and asset-allocation strategies.
> Reverse-engineered: 2026-10-30; data as-of date observed on site: 2026-10-29.

---

## 1. Global Architecture & Navigation Tree

### 1.1 Tech stack & architecture (reverse-engineered findings)

- **Frontend**: React 18 SPA (Vite build, client-side routing; per-page lazy-loaded JS chunks such as `assets/CbBoard-KJi2ArGM.js`). UI kit is shadcn/ui + Radix (Radix behavior signatures: `data-slot=popover-trigger`, `role=tab`). Charts are ECharts (`<div>` containers + canvas; legends/axes match ECharts conventions). Dark/light theme toggle (class-based, top-bar button).
- **Data layer (key architectural fact)**: apart from auth, there is **no dynamic backend API**. All market/research data ships as **build-time static JSON** under `/data/**.json`, lazily fetched by the frontend per page. Data files carry an `updated` field (data as-of date).
- **Auth**: the only dynamic endpoints are `/api/auth/login` (POST) and `/api/auth/me` (GET, called on every page load). The login screen renders at `/` when unauthenticated, with Login/Register tabs (registration requires admin approval; password ≥8 chars with uppercase + digit). All content is gated; unauthenticated visits to any route redirect to the login page.
- **Source layout (from `code-path` attributes left in the build)**: `src/pages/*.tsx` (pages), `src/sections/*.tsx` (business sections: CbDashboard, CbChipBond, SiteHeader, SiteFooter…), `src/components/*.tsx` (shared: AnchorNav, ChartRangePicker, StrategyNavChart, CandleChart, FeedbackWidget, GuideWidget, ui/table…).
- **Global floating widgets**: bottom-right "Feedback" (FeedbackWidget) and "Site Guide" (GuideWidget — an NL Q&A assistant offering deep links).
- **Footer**: ICP filing 京ICP备2026063046号.

### 1.2 Site map (all routes)

```
/                         Home = Login page (unauth) / Daily Brief (authed)
/cb                       Convertible bonds hub (6 module cards)
  /cb/segment             CB segmentation (12 dimensions)
  /cb/clause              Clause tracking (5 tabs)
  /cb/chips               Chip distribution (3 tabs)
  /cb/board               Bond master board (48-column table)
  /cb/quant               CB quant strategies (3 strategies)
  /cb/market              → redirects to /cb/market/overview
    /cb/market/overview   Market overview (scale / indices / supply)
    /cb/market/valuation  Valuation (metrics / prem100 / turnover / abnormal)
    /cb/market/greeks     Greeks & attribution (+ monthly review articles)
    /cb/market/holders    Investor structure (CB ETFs + exchange holder mix)
  /cb/bond/:code          Bond detail page (e.g., /cb/bond/110081.SH)
/eb                       Exchangeable bonds hub (2 module cards)
  /eb/board               EB master board (485 bonds incl. delisted)
  /eb/market              EB market tracking (private/public tabs)
  /eb/notes               EB research notes (local-environment only)
/ps                       Preferred stock hub (2 live modules + 1 pending)
  /ps/board               PS master board (29 issues)
  /ps/market              PS market tracking
/fund                     Funds hub (3 groups, 10 sub-modules + 2 pending)
  /fund/hf                FI+ fund high-frequency board (2,141 funds)
  /fund/market            FI+ fund market aggregates
  /fund/shelf             FI+ manager shelf distribution (143 managers)
  /fund/strategy          FI+ fund strategies (indices / enhanced FOF / draw-the-line)
  /fund/pb                Pure-bond fund tracking board (1,674 funds)
  /fund/pb-duration       Pure-bond duration tracking (Kalman)
  /fund/pb-shelf          Pure-bond manager shelf (157 managers)
  /fund/lowvol-strategy   Pure-bond fund indices (benchmark + enhanced)
  /fund/bond-market       Bond-fund market composite board
  /fund/pension           Pension product tracking (RBSA)
/etf                      ETF hub (4 live + 1 pending)
  /etf/board              ETF per-fund board (1,698 ETFs)
  /etf/scale              ETF market structure
  /etf/pb                 Pure-bond ETF board (51 ETFs)
  /etf/stock-etf          Equity ETF strategies (3 momentum strategies)
/strategy                 FI+ strategies hub (2 modules)
  /strategy/income-equity Dividend-enhanced equity strategy
  /strategy/multi-asset   Multi-asset allocation lab (stock-bond, 44 param sets)
/about                    About (site changelog)
```

### 1.3 Global controllers (state / filters)

| Controller | Scope | Default | Options |
|---|---|---|---|
| Theme | Whole site | dark | dark / light (persisted) |
| Auth session | Whole site | — | `/api/auth/me`; fail → login redirect |
| **ChartRangePicker (time window)** | per-chart component, uniform UI site-wide | `All` | `All / 5Y / 3Y / 1Y / YTD / custom start date (YYYY-MM-DD)` |
| **Download** | per chart | — | exports current-view CSV (some tables: "monthly CSV", "detail CSV") |
| Weighting scheme | /cb/segment, index charts | equal-weight | equal-weight / market-cap-weighted (top button group) |
| Theme-linked chart palette | all ECharts | follows theme | colors switch with theme |

> There is no cross-page global time filter or asset switcher; time windows are **component-level** state. Sub-navigation (e.g., the 4 tabs under /cb/market) is page-level.

### 1.4 Shared component specs

1. **AnchorNav**: sticky "on this page" nav; buttons scroll to sections.
2. **ChartRangePicker**: button group (All/5Y/3Y/1Y/YTD) + custom-start date input + Download. On change: NAV charts re-base inside the window (first value = 100 or 1000); performance tables recompute client-side.
3. **KpiCard**: label + value mini-cards.
4. **DataTable**: sortable column headers, sticky header, row links (bond code → detail page), pagination (50/page).
5. **Tooltip**: ECharts crosshair + series/date/value.

---

## 2. Page-by-Page Breakdown

### 2.0 Home — Daily Brief (URL: `/`)

**Purpose**: auth gate + daily morning brief (3–5 market bullets + tiered risk alerts), updated every trading day from the site's own data.

**Components**:
- **Hero animation**: single-play — line-art keyboard → typewriter "Luca Luo · Daily Brief" → hold.
- **Card_DailyBrief**: `date` + "latest" badge + summary cards, each: `tag` (module tag: CB / Fund / …) + `text` (≤300 chars) + deep `link`.
- **Card_RiskSignals**: tiered alerts (level: high/middle/low + text + link); empty when none.
- Disclaimer: "For research reference only; not investment advice."

**Source**: `GET /data/daily_brief.json`
```json
{
  "date": "2026-10-29",
  "items": [{ "tag": "转债", "text": "CSI CB index −1.13% to 1600.1 …", "link": "/cb/market/overview" }],
  "risks": [],
  "auto": {
    "cbDate": "2026-10-29", "cbN": 319, "cbMv": 5967.95, "cbMvChgPct": -1.12,
    "cbpMed": 59.34, "ytmMed": -3.53, "prem100": 41.78, "prem100Pctile": 93.1, "cbpMedPctile": 97.5,
    "abnPct": 1.88, "turnMed": 4.21, "zzcb": 1600.1, "zzcbChg": -1.13,
    "cbMvIdx": 1616.3, "cbMvIdxChg": -1.09,
    "callExecYtd": 89, "callNoexecYtd": 107, "resetExecYtd": 72, "resetNoexecYtd": 300,
    "etfDate": "2026-10-29", "etfScale": 50414.7, "etfScaleChgW": 1049.2, "etfBondScaleChgW": 269.8,
    "riskSignals": [{ "level": "high", "link": "/cb/market", "text": "Par-100 premium 41.78%, 93.1th pctile of past 3Y" }]
  }
}
```
**auto semantics**: cbN = number of live CBs; cbMv = CB market cap (¥100M); cbpMed = median conversion premium (%); ytmMed = median YTM (%); prem100 = par-100 premium (%); prem100Pctile = its 3Y percentile; abnPct = abnormal-bond share (%); turnMed = median turnover (%); zzcb = CSI CB close; cbMvIdx = cap-weighted CB index; call/reset Exec/Noexec YTD = redemption/reset announcement counts; etfScale = total ETF AUM (¥100M); etfScaleChgW = weekly change.

---

### 2.1 Module: Convertible Bonds (`/cb`)

#### 2.1.1 /cb hub — 6 module cards (status badge "Live" + h2 + one-line desc + "Enter"): segment / clause / chips / board / quant / market.

#### 2.1.2 Page: Market Overview (URL: `/cb/market/overview`)

**Purpose**: long-run tracking of CB market scale, count, index performance and supply structure.

**Components** (top to bottom):

1. **Chart_CBScale** (two charts)
   - A: dual-axis lines (L 0–12,000 ¥100M): `outstanding balance` & `outstanding market cap`; headline "Latest: balance 473.8bn · cap 596.8bn · 319 bonds".
   - B: single line `live count` (0–600).
   - X: monthly (from 2004-10).
2. **Chart_CBIndex** ("monthly rebalance, base 1000")
   - Headline: "Latest: CB equal-weight 2272 · CB cap-weighted 1616 · stock equal-weight 1707 · stock cap-weighted 1462".
   - 5 lines (legend-clickable): `CSI CB / stock cap-weighted / stock equal-weighted / CB cap-weighted / CB equal-weighted`.
   - **Table_IndexPerf**: columns Index | Period cumulative return | Annualized return | Annualized vol | Max drawdown; 5 rows.
   - Methodology (verbatim): CB EW/CW indices hold public CBs (balance > ¥30mn, excluding those with triggered redemption/delisting), monthly rebalance, coupon-adjusted (AdjPrice); stock legs are 1:1 matching stock portfolios of the same holdings (multiple CBs per stock take multiple slots), using backward-adjusted closes; CSI CB is a price index for reference. NAVs re-based to 1000 at window start; stats computed on the selected window; annualization uses 243 trading days.
3. **Section_CBSupply**
   - A **New issuance**: bars (issue size ¥100M) + line (count, right axis).
   - B **Delisting**: stacked bars (converted-out size / other-exit size) + line (count).
   - C **Net supply**: bars = monthly net supply (red = increase / green = decrease) + line = rolling 12M cumulative. Formula "new issuance − delisting (issue-size basis)".
   - **Table_YearlySupply** (2010–2026): year | issue_n | issue_sz | conv_delist_n | conv_delist_sz | other_delist_n | other_delist_sz | net_n | net_sz | digested-by-conversion | digested-by-redemption. Note: "conversion digested = principal converted during life; redemption digested = final cash redemption (issue size = the two combined)".

**Sources**:
- `GET /data/cb_market.json` → `{updated, daily:{cal:string[2397] (YYYYMMDD, 20041019→), rows:number[2397][28]}, holder:{sse,szse}}`. **rows is a 28-column compact array** (verified column by column):
  ```
  [0]  live count N
  [1]  outstanding balance (¥100M)
  [2]  outstanding market cap (¥100M)
  [3]  conversion value · median   [4] · mean   [5] · cap-weighted
  [6]  bond floor value · median   [7] · mean   [8] · cap-weighted
  [9]  remaining maturity (y) · median [10] · mean [11] · cap-weighted
  [12] YTM (%) · median  [13] · mean  [14] · cap-weighted
  [15] conversion premium (%) · median [16] · mean [17] · cap-weighted
  [18] bond-floor premium (%) · median [19] · mean [20] · cap-weighted
  [21] par-100 premium (%)
  [22] par-100 regression sample count
  [23] abnormal bond count
  [24] abnormal share (%)
  [25] turnover (%) · median  [26] · mean  [27] · cap-weighted
  ```
  Latest row example: `[319, 4737.7, 5967.95, 81.32, 91.43, 94.79, 106.55, 105.57, 106.6, 2.69, 3, 2.59, -3.53, -9.65, -9.06, 59.34, 73.55, 63.56, 19.83, 36.37, 30.46, 41.78, 293, 6, 1.88, 4.21, 16.14, 8.64]`
- `GET /data/cb_idx.json` → `{updated, cal:string[2140] (YYYYMMDD, 20171229→), series:{CB_EQ:[], CB_MV:[], STK_EQ:[], STK_MV:[], ZZCB:[]}}` (daily index points, monthly rebalance, base 1000).
- `GET /data/cb_supply.json` → `{updated, monthly:[202]{m:"2010-01", issue_n, issue_sz, conv_n, conv_sz, other_n, other_sz, dig_conv, dig_cash}, yearly:[17]{y, …}}`.

#### 2.1.3 Page: Valuation (URL: `/cb/market/valuation`)

1. **Tabs_ValuationMetrics**: 6 tabs — `conversion value / bond floor / maturity / YTM / conversion premium / bond-floor premium`. Each tab: 3 KPIs (median / mean / cap-weighted) + 3-line chart (same names). Units: value metrics in yuan; maturity in years; YTM & premiums in %.
2. **Chart_Prem100 (par-100 premium, regression-fitted)**: single line (%). Headline "Latest 41.78% · regression sample 293".
   - Formula (verbatim): daily cross-sectional regression y = a·e^(−b·x) + c where y = conversion premium, x = conversion value / 100; read the fit at x = 1 (par 100). Valid sample: bonds without announced forced redemption and remaining maturity > 0.5y.
3. **Chart_Turnover**: 3 lines (median/mean/cap-weighted, %). "Turnover = daily amount / total market cap; cap-weighted by each bond's market cap."
4. **Chart_Abnormal**: titled "Abnormal bonds (price ≥130 AND conversion premium ≥30% AND turnover ≥100%)"; bars (count) + line (share %, right axis).

#### 2.1.4 Page: Greeks & Attribution (URL: `/cb/market/greeks`)

Header methodology: "delisting-adjusted BS model; discount rate = bond-floor YTM implied from pure bond value; delisting line = max(1/stock price, ¥300mn/total cap) × parity; expected return = delta·s·rf + ½·gamma·(s·σ)² + theta (annualized); attribution uses full revaluation."

**Components**:
1. **Chart_MarketDelta**: 3 lines (median/mean/cap-weighted, L axis 0–1) + right-axis sample count. Headline "median 0.681 · mean 0.655 · cap-weighted 0.647 · 308 bonds".
2. **Chart_ExpRet (expected return, % ann.)**: 3 sub-tabs:
   - Three lines: median / cap-weighted / mean.
   - Decomposition: stacked "expected return = delta drift + gamma vol + theta decay" (delta/gamma stack up, theta down; light line = mean total).
   - IV comparison: median implied vol vs median 120d realized vol of underlying stocks (%).
3. **Chart_AboveYTM**: line (share %, L) + bars (count, R). "Share = # bonds with E[ret] > own bond-floor YTM / valid sample."
4. **Table_AboveYTMLatest**: code(link) | name | close | parity | E[ret]% | floor YTM% | gap%, sorted by gap desc (32 rows).
5. **Chart_Attribution (monthly, full-revaluation)**: stacked bars (delta/gamma/rho/theta/vega/residual; positives stacked above 0, negatives below) + light line = actual return.
6. **Accordion_Explain**: full model documentation (BS formula, 5-factor table, long-run reference 2017-02~2026-09, 116 months: actual +0.69 = delta +0.49 + gamma +0.25 + theta −0.04 + vega −0.02 + rho +0.06 + residual −0.05).
7. **Section_MonthlyReview**: combobox of report months (from `/data/cb_reports/index.json`, 2021-01~2026-10, 70+ issues; draft=true shows a "draft" badge); body is a structured long-form article (Market overview / Attribution & weekly rhythm / Valuation & sentiment / Segment structure / Bond-level color / Supply & clauses / Demand & liquidity) + an attribution table (period | actual | delta | gamma | theta | vega | rho | residual | n, incl. MTD total row).

**Core calculation logic**:
- **Theoretical value**: `V = survival_prob × [ parity s·N(d1) + redemption-at-maturity X·e^(−r·t)·(1 − N(d2)) ]`; s = conversion value; t = maturity; X = maturity redemption price; r = bond-floor YTM; σ = implied vol solved from market price; survival prob = probability the stock never crosses the delisting line (max(1/price, ¥300mn/total cap) × parity) within t.
- **Delta**: `Δ = ∂V/∂s = N(d1)·survival_prob + delisting correction`.
- **Expected return**: `E[ret] = ( Δ·s·r_f + ½·Γ·(s·σ_his)² + Θ ) / P`; r_f = risk-free (site assumes stock expected annual return = 4%).
- **Monthly 5-factor attribution (v2, full revaluation)**: `month return = delta + gamma + theta + vega + rho + residual`; each step re-prices the model after swapping in one new parameter (s → σ → t → r); the price gap is that factor's exact contribution.

**Source**: `GET /data/cb_greeks.json`
```jsonc
{
  "updated": "2026-10-29",
  "delta":   [{ "d": "2017-01-03", "median": 0.669, "mean": 0.687, "mv": 0.69, "n": 15 }, …],  // 2383 days
  "exp_ret": [{ "d": "…", "median": 0.23, "mean": -0.147, "mv": 0.697, "n": 298,
                "iv": 68.725, "vol_his": 48.383,          // median IV / median realized vol (%)
                "d_term": 1.624, "g_term": 1.411, "t_term": -3.096 }],   // decomposition (%)
  "above_ytm": { "series": [{"d","n","pct"}], "latest_date": "…",
                 "latest": [{"code","name","close","conv_v","exp_ret","ytm","gap"}] },  // gap-desc
  "attribution": [{ "m": "2017-02", "n": 15, "ret_actual": 0.341, "delta": 2.045, "gamma": 0.061,
                    "theta": -0.016, "vega": -1.82, "rho": 0, "residual": 0.071 }, …]  // 117 months
}
```
Reports: `/data/cb_reports/index.json` (month index), `/data/cb_reports/{YYYY-MM}.json` (title+body), `/data/cb_reports/explain.json`.

#### 2.1.5 Page: Investor Structure (URL: `/cb/market/holders`)

1. **Section_CBETF**: exactly two funds — Boshi CB ETF (511380.SH) and Haitong CB ETF (511180.SH). Headline "combined AUM 70.48bn · 11.43% of CB (incl. public EB) market".
   - **AUM & shares**: sub-tab "AUM / shares"; 3 lines (Boshi / Haitong / combined). "AUM = total shares × close; shares from exchange disclosure (recorded on change dates, forward-filled between trading days)."
   - **Share of CB market**: 3 lines (%). "Denominator = CB total cap + public EB total cap (both ETFs hold public EBs); public EB cap = close (ffill) × outstanding face value."
   - **Flows**: sub-tab "net subscription estimate / main-force net inflow"; stacked bars (per-fund daily net subscription) + line (combined 20-day cumulative, right axis). "Net subscription = Δ shares × close."
2. **Section_HolderStructure**: SSE/SZSE tabs; sub-tab "holdings (face value ¥100M) / trading (amount ¥100M)"; window combobox (~3Y).
   - 100%-stacked area chart by investor type + **Table_LatestMonth**: investor type | face value (¥100M) | share %.
   - SSE 17 investor types: public funds, enterprise annuities, retail, securities proprietary, general institutions, private funds, insurers, securities AMCs, fund-company AUM plans, social security fund, QFII & RQFII, trusts, bank wealth management, other financial institutions, bank proprietary, finance companies, Shanghai connect investors.
   - SZSE 14 types: funds, enterprise annuities, retail, general institutions, private funds, broker proprietary, insurers, other professional institutions, broker collections, fund segregated accounts, social security, trusts, RQFII, QFII.

**Source**: `GET /data/cb_etf.json`
```jsonc
{
  "updated": "2026-10-29",
  "etfs": [{ "code": "511380.SH", "name": "…", "short": "博时可转债ETF" }, { …511180.SH }],
  "cal": ["20200407", …],              // 1591 days
  "cbMv": [4426.31, …], "ebMv": [1081.03, …],      // CB cap / public-EB cap (¥100M)
  "series": { "511380.SH": { "shares":[], "mv":[], "flow":[first null], "mflow":[], "amt":[] }, "511180.SH": {…} }
}
```
Holder matrices inside `cb_market.json → holder`:
```jsonc
"holder": {
  "sse": { "months": {"0":"2016-07",…}, "investors": {"0":"公募基金",…17},
           "hold":  {"<investor_idx>": {"<month_idx>": 125.74|null}},    // face value ¥100M
           "trade": {…same shape}, "holdUnit": "面值(亿元)", "tradeUnit": "成交金额(亿元)" },
  "szse": { "months": {"0":"2019-05",…}, "investors": {…14}, "hold": {…market value ¥100M}, "trade": {…} }
}
```

#### 2.1.6 Page: CB Segmentation (URL: `/cb/segment`)

**Purpose**: group CBs along 12 dimensions and compare segment indices, membership counts, market caps, and current holdings.

**Global controls**: weighting (equal / cap) + dimension tabs ×12 + ChartRangePicker.

**Full 12-dimension definitions (verbatim from dims metadata)**:

| key | label | Grouping rule |
|---|---|---|
| style | equity-credit character | conversion value / bond floor: <0.8 debt-like; 0.8–1.2 balanced; >1.2 equity-like |
| price | price | close <110 low; 110–135 mid; >135 high |
| industry | SW industry | underlying's SW-2021 level-1 industry |
| major | style mega-group | SW L1 industries merged into 7 mega groups (cyclical / equipment mfg / tech / staples / discretionary / stable / big financial) |
| maturity | maturity/lifecycle | 6 mutually exclusive groups: new (listed ≤2 months); sub-new (pre-conversion-window); regular (>2y left); old (1–2y); near-window (1 month–1y); endgame (redemption announced or ≤1 month) |
| rating | bond rating | high AAA/AA+; mid AA/AA−; low A+/A; weak ≤A− (incl. defaulted); unrated excluded; PIT by rating date |
| size | issue size (static) | ≥5bn large; 2–5bn mid-large; 0.5–2bn mid-small; <0.5bn small |
| stk_mv | underlying market cap | on rebalance day: ≥50bn large; 20–50bn; 10–20bn; 5–10bn; <5bn small |
| fund_hold | fund ownership | public-fund holdings / bond market cap (quarterly/semi/annual reports, A/C dedup, PIT by announce date): ≥50% heavily held; 30–50%; 10–30%; <10% lightly held; pre-conversion-window bonds in a separate "not yet convertible" group; no disclosure within window treated as 0 (light group) |
| doublelow | double-low | on rebalance cross-section: price rank + premium rank = double-low score; quintiles Q1 (most double-low) … Q5 |
| stk_vol | underlying volatility | 120-day return std (adjusted close) quintiles Q1 (lowest) … Q5 |
| stk_mom | underlying momentum | 55-day cumulative return (adjusted close) quintiles Q1 (weakest) … Q5 (strongest) |

**Per-tab component layout (uniform)**:
1. **Table_SegmentPerf**: rows = groups + "full-sample benchmark"; leading columns: latest level (base 1000), member count; column groups: last 1W (with dynamic start date) / 1M / QTD / YTD / 3Y, each with return% / maxDD% / ann.vol%.
2. **Chart_SegmentNav**: group NAV lines + dashed benchmark; re-based to 100 at window start; header lists "base 2016-12-30 = 100 · window returns: …".
3. **Chart_SegmentCount**: stacked cumulative membership count; dashed total.
4. **Chart_SegmentMv**: stacked cumulative market cap (¥100M); dashed total.
5. **Table_CurrentHoldings**: group combobox; caption "rebalance 2026-09-30 · 124 members · price/size as of 2026-10-29 · equal-weight in group"; columns: code(link) | name | close | equity-credit ratio | outstanding (¥100M) | SW industry | mega style.

Rebalance rule: month-end rebalance, base 1000.

**Sources**:
- `GET /data/cb_segment_idx.json` → `{updated, dims:[12]{key,label,desc}, groups:{<dim>: {<group>: {s,c,n,mv}}}, bench:"全样本", cal:[2384]}`.
- `GET /data/cb_segment_idx_style.json` → full series for the style dimension.
- `GET /data/cb_segment_hold.json` → `{updated, holds:{<group label>: {rebal:"2026-09-30", n:124, items:[[code,name,close,style_ratio,size_yi,industry,major], …]}}}` (60+ group labels).

#### 2.1.7 Page: Clause Tracking (URL: `/cb/clause`)

**Tab 1 · Conditional redemption**: yearly bar chart (red = redemption executed / gray = announced no-redeem), 2013–2026; **Table_Call** (319 bonds) with search (code/name/stock) + status combobox. Columns: code(link) | name | stock | CB close | conversion value | conv price | maturity (5.1y) | trigger progress (26/15 = days hit / days needed) | status (`triggered / counting / exempt`) | waiver / event.

**Tab 2 · Conversion-price reset**: yearly bars (blue = reset proposed / gray = no-reset); same table with "latest reset / event" column (e.g., "proposed, price TBD" / "reset to 6.18 (2025-03-27)"); statuses `proposed-pending / triggered / counting`; progress may show "counting paused".

**Tab 3 · Conditional put**: table with progress "167 consecutive days", status "condition met", note, and days-to-put-window.

**Tab 4 · Yearly statistics**: yearly chart + per-year lists for redemption (89) and reset (67) events; columns: code | name | announce date | days to delisting | prior clause history ("triggered N times / no-redeem N / redeemed N").

**Tab 5 · Event returns**: "returns 5/10/20 trading days after announcement (base date = announcement date, incl. same-day move; CB excess vs CSI CB index; stock excess vs CSI 800 total return)". Controls: yearly/quarterly; metric (CB return / CB excess / stock return / stock excess); window. 4 mini yearly-mean charts (reset-exec / reset-noexec / call-exec / call-noexec). **Table_EventStats**: year | reset samples (total/exec/noexec) | call samples | per-event (mean% / win-rate% / payoff) × 4 categories, 2013–2026.

**Sources**:
- `GET /data/clause_board.json` → `{updated, call:[319], reset:[319], put:[319]}`, record fields:
  ```
  code, name, stk_name, cb_close, convv, conv_price, ptm,
  ratio (trigger %: 130/85/70), window (30), need (15 or 30),
  cnt (days counted, null if n/a), gap (days to trigger), status (triggered|counting|proposed|…),
  exempt_end, last_event, last_reset:{date,price}, note, days_to
  ```
- `GET /data/clause_events.json` → `{updated, counts:{yearly:{call:[{period,exec,noexec}], reset:[…]}, quarterly:{…}}, stats:[{period, event:"下修"|"赎回", executed:bool, n, metrics:{cb_ret_5d/10d/20d, cb_excess_*, stk_ret_*, stk_excess_*: {mean, win, payoff, n}}}]}`.

#### 2.1.8 Page: Chip Distribution (URL: `/cb/chips`)

**Tab 1 · Single-bond chips**: cmdk bond picker ("110081.SH 闻泰转债 电子"); header card; **price chart (OHLC + volume + chip-weighted price line, amber)**; **chip factor (%) = close / chip-weighted price − 1** (e.g., −12.75%); **chip distribution histogram** (date-selectable; green = in-profit chips below close / red = trapped chips above close; caption "close 90.27 · weighted 103.47 · in-profit 1.5%").

**Tab 2 · Market chips**: market chip factor (%; red cap-weighted / gray equal-weight, from 2014-12); underwater share (close below cost) cap-weighted, 0–100%; **industry chip factor cross-section** (SW L1 horizontal bars, −18%~6%; cross-section/history toggle; clicking an industry shows its cap-weighted history vs market).

**Tab 3 · Fund ownership by segment**: "within-dimension multi-select = OR; across dimensions = AND; labels are PIT report dates." 12-dimension tag groups + "convertible-window only" + start date (default 2017-03-31) + metric toggle (cap-weighted % / exclude CB ETFs / market-wide comparison). Trend chart + **Table_FundHoldPeriods**: report date | count | cap-weighted % | EW mean % | median % | holdings ¥100M | market cap-weighted % (39 periods).

**Sources**:
- `GET /data/chips_index.json` → `{updated, bonds:[1009]{code,name,stk,ind,listed}}`.
- `GET /data/chips_market.json` → `{updated, market:[[date, f_eq, f_mv, ?, ?, profit_ratio]×2885], industries:[29]{name, factor, profit, n, mv}, industryDaily:{<ind>: [[date, f_mv], …]}}`.
- `GET /data/chips/{code}.json` → `{code, name, daily:[[date, close, weighted_price, factor, profit_ratio]×1255], dist:{"<date>": [[bin, pct], …]}}`.
- `GET /data/chips/fund_hold.json` → `{updated, periods:[95], codes:[1079], names:[1079], dims:{12 dims→group labels}, dimOrder:[12], rows:[13388] compact [p_idx, c_idx, cap_w%, ew%, med%, mv, …]}`.

#### 2.1.9 Page: Bond Master Board (URL: `/cb/board`)

- **KpiRow**: filtered count / live / delisted / live balance total (¥100M).
- Status toggle: live / delisted / all. Search: "code / name / stock". Column settings (26/48 shown) + reset + filter builder (field + op gt/lt/contains + value).
- **Table_Board**: default 26 columns: code(link) | short name | status | type (CB/EB) | stock code | stock name | exchange | issue size (¥100M) | outstanding (¥100M) | latest conv price | initial conv price | call trigger price | reset trigger price | put trigger price | coupon (%) | compensating coupon (%) | tenor (y) | par | issue price | maturity redemption price | coupons per year | issue rating | latest rating | list date | delist date | maturity date.
- Selectable columns (full key list, 57 keys): call_ratio / reset_ratio / put_ratio (%)… has_call / has_put / call_start / call_end / put_start / put_end / call_days / call_max_days / put_days / put_max_days / reset_days / reset_max_days / rate_type / value_date / conv_start_date / conv_end_date / guarantor / guarantee_type / rating_comp / rate_clause / call_clause / put_clause / reset_clause / conv_clause.
- Pagination 50/page; column sort; nulls render "—".

**Source**: no separate file — the **static basic-info dataset (1,095 rows) is inlined into the CbBoard JS chunk at build time** (dataset_name "转债静态基础信息", last_updated 2026-10-29 22:41). Raw record fields (Tushare-style naming) listed in §3.3. Front-end derivations: `status = delist_date ? delisted : live`; sizes ÷1e8 → ¥100M; date reformat. Client-side filter rule (from bundle): drop `cb_type==="EB"`, drop codes prefixed 1108/1185/1240/1459/1450/1171/1210 (detachable warrants), drop rows lacking both list_date and value_date.

#### 2.1.10 Page: CB Quant Strategies (URL: `/cb/quant`)

Header: "backtest: equal-weight, coupon-adjusted, 0.1% one-way cost."

1. **StrategyCards** ×3 — name + style badge (offense/balanced/defense) + rebalance frequency + rule text + 4 KPIs (count / ann. return / ann. vol / maxDD) + status line ("−9.2% from peak · 120 days without new high · inception 2018-12-28").
   - **Priced High-Vol (offense, weekly)**: "BS absolute pricing (forecast vol) undervalued top 25%, then top 50% by forecast vol" (38 bonds, 28.0% ann.)
   - **Convexity Opt. (balanced, monthly)**: "gamma×vol²/price + 60d momentum score, industry-neutral top 1/3, monthly" (21, 15.7%)
   - **Defensive Pick (defense, weekly)**: "rating ≥AA− and maturity ≥0.5y, cheapest 30% by price in pool" (63, 12.6%)
2. **Table_PerfByRange**: rows 1W/1M/3M/YTD/1Y/3Y/full (annualized); columns = 3 strategies + CSI CB; each cell return% + sub-line (maxDD −x% · vol x%).
3. **Chart_Nav**: 4 lines; "window start 2018-12-28 · each series re-based to first non-null NAV = 100 · window returns: …".
4. **Section_Holdings**: strategy tabs; two tables — **Actual holdings** ("rebalance 2026-10-23 · N bonds · weight desc") and **Simulated rebalance** (recomputed at latest trade date); columns: code | name | weight(%) | price | conversion premium(%) | rating | industry.

**Sources**: `/data/cb_strategy/metrics.json` `{updated, strategies:[{策略,style,rebalance,desc,起始日期,近1周:{收益,最大回撤,波动率},…,全区间}×4]}`; `nav.json` `{updated, names:[4], series:[[date, nav×4]×1897]}`; `holdings.json` `{updated, holdings:{<strategy>:{actual:{换仓日期,持仓:[{代码,名称,权重,价格,转股溢价率,评级,行业}]}, simulated:{…}}}}`.

#### 2.1.11 Page: Bond Detail (URL: `/cb/bond/:code`)

1. **Header**: name+code, status badge, latest rating badge, stock (link), full legal title, value/list/maturity dates.
2. **KpiGrid ×8**: latest close (+date) | conversion premium % | bond-floor premium % | YTM % | bond floor value | conversion value | outstanding (¥100M) | maturity (y).
3. **Candle chart (OHLC + volume)**; **premium chart** (conversion & bond-floor premium); **value comparison** (close / conversion value / bond floor); **outstanding size & changes** + table (date | cause issue/convert/put | size); **rating history** (date | rating | action downgrade/affirm/first | prior); **issue & terms summary** table; **accordion with full clause texts ×4**.

**Source**: `GET /data/bonds/{code}.json`
```jsonc
{ "code": "110081.SH",
  "basic": { ts_code, bond_full_name, …, conv_price, issue_size, … },      // same fields as board dataset
  "daily": [[ "d":"2021-07-28", "strbv":89.96 /*bond floor*/, "strbp":11.16 /*floor prem %*/,
              "convv":95.43 /*parity*/, "convpr":4.79 /*conv prem %*/, "iv":null /*IV %*/,
              "convp":96.67 /*close*/, "rt":"AA+" /*rating then*/, "osz":860000 /*outstanding ¥*/,
              "dil":7.15 /*dilution %*/ ] × 1272 ],
  "events": { "credit":[…], "shares":[…] } }
```

---

### 2.2 Module: Exchangeable Bonds (`/eb`)

#### 2.2.1 /eb/board — filters (status / public-vs-private / rating); "485 bonds (59 live / 426 delisted)". **Table_EBBoard**: code | name | stock | public/private | status | close | conversion value | conv price | pure-bond YTM% | maturity (y) | issue size (¥100M) | outstanding (¥100M) | rating | issue type | board (main/STAR/ChiNext) | list date; no-quote names show "—".
**Source**: `GET /data/eb/eb_board.json` → `{updated, px_date, rows:[485]{code,name,stk,stk_name,issue_type,mkt,board,list_date,value_date,delist_date,maturity,term_y,scale,outstanding,close,close_date,conv_price,convv,ytm,ptm,rating,rating_chg,live}}`.

#### 2.2.2 /eb/market (tabs private/public)
- **Market totals**: live count + traded-today count; outstanding face value (L) + turnover (R).
- **EB index** ("CSI-valuation priced, monthly rebalance, base 1000"): 6 lines — private EB EW/CW, stock EW/CW, private EB EW/CW (trade price). Membership count + holdings cap chart + performance table. Verbatim: "private EBs (listed, balance > ¥30mn, CSI valuation available that day), monthly rebalance, CSI-valuation pricing with coupon adjustment; dashed lines price the same holdings at exchange closes (ffill if untraded); stock legs 1:1 matching portfolios; annualization 243 days."
- **Single-bond trading**: 20-day turnover top-20 bars; per-bond close (L) + daily amount (R) selector.
- **Outstanding structure**: rating bands / price bands (90-100 / 100-110 / 110-120 / untraded) / issue-size bands (2-5 / 5-10 / 10-20 / ≥20 ¥100M) / issue-type; each shows face value + count.
- **Clause & lifecycle events**: type filters (all 465 / redemption 31 / reset 68 / conversion 85 / maturity 93 / early redemption 10 / issuance 175); table: date | code | name | type | detail ("reset effective, conv price 72.63", "redem", "conv", "mature", "issue", "no-redeem announced"); top 100, date-desc, includes scheduled future events.
**Sources**: `/data/eb/eb_market.json`, `/data/eb_idx.json`, `/data/eb/detail/{code}.json`.

#### 2.2.3 /eb/notes — five-chapter research notes; content served only in local environment.

---

### 2.3 Module: Preferred Stocks (`/ps`)

#### 2.3.1 /ps/board — filters (all / bank / other-listed / OTC-board; live). 29 rows. **Table_PSBoard**: code | name | segment | company | list date | size (¥100M) | dividend rate ("3.48 floating") | CSI valuation | yield % | modified duration | close | last dividend (amount+date) | rating | clause badges (convert/force-convert/put/cumulative/redeem). Note: only SH/SZ quotes exist; .NQ board shows CSI valuation.
**Source**: `GET /data/ps/ps_board.json` → `{updated, px_date, rows:[103]{code,name,seg(bank/listed/nq),company,live,list_date,delist_date,scale,div_rate,rate_type,close,close_date,csi_val,ytm,dur,csi_date,last_div,last_div_date,rating,rating_date,conv,force_conv,put,cum,redeem}}`.

#### 2.3.2 /ps/market — snapshot KPIs ×4 segments (live count / size / median yield / weighted dividend / median duration); outstanding trend (stacked by segment, monthly); yearly supply (+ issuance / − retirement); weekly yield & duration by segment (CSI-valuation median; footnote about exercise-scenario jumps and OTC high yields from put-window discounting); dividend-rate distribution 1-2…8-9 and fixed/floating; **Table_TermStruct** clause × segment share% (convertible 58.6 / force-convert 62.1 / put 34.5 / cumulative 37.9 / mandatory-pay 3.4 / voting 10.3 / redeemable 100.0); annual dividends (¥100M, L) + count (R); **event stream** (24 months; dividend/new/delisting with per-10-share amounts).
**Source**: `GET /data/ps/ps_market.json`.

---

### 2.4 Module: Funds (`/fund`)

Hub groups: FI+ funds / pure-bond funds / market watch (10 modules + 2 pending).

#### 2.4.1 /fund/hf — FI+ high-frequency board
- KPIs: 2,141 funds / ¥3,783.7bn AUM / primary bond 431 / secondary bond 793 / hybrid 868 / CB funds 49.
- **Multi-dimensional filter panel** (AND across groups, OR within): category ×4; equity status; AUM bands (<2 / 2–10 / 10–50 / ≥50 ¥100M + custom); institutional holding (<20 / 20–50 / 50–80 / ≥80% + custom); disclosed duration (<1 / 1–2 / 2–3 / 3–5 / ≥5y + custom); HF-estimated stock position (0 / 0–5 / 5–10 / 10–20 / ≥20% + custom); CB position (0 / 0–10 / 10–30 / 30–50 / ≥50% + custom); performance percentile (3M/6M/12M × return/drawdown × quartiles + absolute thresholds); shelf tags (trading/non-trading/position-trading/style-trading/value-enhance/growth-enhance/carry-enhance/CB-strategy/other/unclassified); volatility tier (low/mid/high/CB fund).
- **Table_HFBoard** (configurable columns; defaults): code | name | category | manager | AUM | 5-day return | disclosed stock position | disclosed CB position. 50/page (43 pages).
- Methodology highlights: sample = four nominal FI+ categories (primary/secondary bond; hybrid = conservative hybrid + flexible funds with rolling 2-quarter equity ≤45%) ∪ current actual-holding CB-fund members; exclude CB ETFs and funds with NAV stale >30 days; stock/CB positions are HF estimates (**kfac model: Kalman-filter regression + holdings mark-to-market, anchored to quarterly disclosures**); MAE ≤3pp good / 3–5pp fair / >5pp weak.

#### 2.4.2 /fund/market — tabs: quarterly holdings (AUM stacked by category + sample count; disclosed stock & CB positions, each median/mean/AUM-weighted; quarterly stats table 2003Q3→2026Q3 with per-quarter median/mean/weighted stock & CB positions); performance distributions (multi-window return & maxDD distribution tables + half-violin; quarterly return grid); HF tracking.

#### 2.4.3 /fund/shelf — 143 managers / 1,640 funds / ¥3,257.5bn. Volatility-tier & strategy-class distribution dual-axis bars; **Table_ShelfByManager**: manager | fund count | AUM | volatility split (low/mid/high/CB) | strategy split | top-2 flagship funds. Tier rule: 4-quarter mean of (stock + CB×0.5) <10% low / 10–20% mid / ≥20% high; CB mean >40% → CB fund. Strategy 3-level: NAV regression vs dedicated indices (R²) splits trading vs non-trading; non-trading further split by holdings grid & style regressions.

#### 2.4.4 /fund/strategy — (1) FI+ fund indices (basic/style × equal/cap-weighted; 5 indices from 2006-12-31; performance & yearly tables); (2) 3-D scored enhanced FOF (pool × strategy balanced/steady/offense/defense/stability × rebalance month/quarter/half-year; NAV + excess NAV vs S0; metrics incl. IR & excess maxDD; dual-caliber holdings with score columns: weight, composite position, score, win-rate, elasticity, no-loss ratio, stock-β CV, bond-β CV; scoring window 250d, market up/down by CSI800 rolling 21d, stability = CV of rolling-60d betas vs CSI800 & CBA20921); (3) draw-the-line portfolio vs money-fund index 885009.WI (monthly new-high probability >0.5 in 243d window and adjusted Euclidean distance ≤0.025; top-10 equal-weight; dual-caliber holdings).

#### 2.4.5 /fund/pb — 1,674 funds / ¥4,651.5bn; category+size filters; table: code | name | category | manager | fund managers | mgmt fee | institutional share | AUM | latest NAV | daily chg | YTD | leverage | duration | estimated duration | NAV date. Sample: pure-bond & near-pure funds with zero stock/CB holdings in latest report, ex closed-ended & non-daily NAVs.

#### 2.4.6 /fund/pb-duration — group duration trends (category × instrument style × metric median/dispersion(75-25)/CV); Kalman daily estimates (ewm5 display smoothing) with semi-annual disclosed dots; latest-read table (sample n, estimated median/mean, dispersion, CV, latest disclosed median, diff); sample-size evolution. Method: quarterly PIT sample; **unconstrained time-varying Kalman regression (burn=60, q_beta=1e-4)** on index set = core10-swapped 5Y local-gov (930865) + 30Y CGB (931080, spliced from 2023-12-19).

#### 2.4.7 /fund/pb-shelf — 157 managers / 2,864 funds / ¥8,491.7bn; quarterly shelf trend (4 dimension toggles: nominal type / duration band / instrument style / amortized) with AUM + share stacked pair; 4 distribution charts; manager table with nominal/duration/style splits and flagship funds.

#### 2.4.8 /fund/lowvol-strategy — pure-bond fund indices: basic/style/leverage/duration-subtype × equal/cap-weighted (short-bond & mid-long pure bond from 2013-03-31; performance & yearly tables) + enhanced indices (core/satellite/layered). Method highlights: category codes 2001010301/2001010302; ex closed-ended & >3M holding periods; purity = zero stock+CB in last ≤4 quarterly reports; style threshold 60% (priority rate→credit→cash-like); leverage index requires rolling-2Q leverage >1.25; duration subtypes from disclosed rate-sensitivity duration over last 4 half-years (min ≥4 long / max ≤3 short / range >2 rotating); size gates ¥200mn/¥500mn.

#### 2.4.9 /fund/bond-market — 9 sections (scale & count / pure-bond product forms / flows / stock position / equity style / CB position / CB style / instrument allocation / leverage). Header methodology: "ΔAUM = NAV contribution + subscriptions/redemptions + new issuance + reclass-in − exits; NAV contribution uses adjusted returns (dividends counted as outflow); holdings metrics for disclosing group only; leverage = total assets/NAV; FI+ groups show pure-bond-core allocation; benchmarks: stocks = all-A cap-weighted (from 2015), CBs = all live CBs cap-weighted." Scale chart: 8 nominal categories × live/new-issue; detail table per report date with 16 value columns + total; CSV download.

#### 2.4.10 /fund/pension — level-1 type equal-weight indices (bond/hybrid/equity/money, base 1000, from 2013-08) + performance table; level-2 type indices (12 types mapped to level-1); fixed-income-ordinary RBSA tracking: weekly stacked position structure (secondary-bond-fund/total pure bond/total equity/total CB/deposits+non-standard/cash; negative cash = leverage), duration = Σ pure-bond weight × index modified duration (p25–p75 band), pure-bond & equity factor breakdowns; **RBSA latest cross-section table** (code | name | manager | pure bond | CB | deposits+non-standard | secondary bond fund | equity | equity anchor | cash | duration | R²) with sorting & CSV. Factor pool verbatim in §2.4.10 of CN edition.

---

### 2.5 Module: ETF (`/etf`)

#### 2.5.1 /etf/board — KPIs (1,698 ETFs / ¥5,041.4bn / daily turnover ¥569.78bn / YTD median −10.47%); category filter (all/equity/QDII/commodity/bond/money); column settings (12/17) + filter builder; default columns: code | name | category | close | AUM (¥100M) | turnover (¥100M) | premium-discount % | 1W % | 1M % | YTD % | 3Y % | YTD maxDD %.

#### 2.5.2 /etf/scale — KPIs (total AUM/count/monthly Δ/2026 new listings); stacked AUM by category (3Y); counts; **monthly ΔAUM decomposition (share contribution vs NAV contribution, 36M)**; monthly new listings by category.

#### 2.5.3 /etf/pb — pure-bond ETFs (CB/EB ETFs excluded); mega-class = credit/rate; 8 instrument types by name & tracked-index keywords. KPIs (51 ETFs / ¥1,037.01bn / mean YTD +2.89% / YTD net share inflow +¥250.28bn); AUM trend stacked by instrument; monthly decomposition; **instrument summary table** (type | count | AUM | share % | mean YTD % | YTD flow). Filters incl. tracked index (24 indices listed). Detail table: 21 columns incl. shares, premium/discount, multi-window returns & maxDDs, duration + duration-as-of, list date.

#### 2.5.4 /etf/stock-etf — "monthly rebalance, equal-weight, adjusted price, zero cost." 3 strategy cards + equal-weight benchmark:
- **Relative momentum (offense)**: "120d momentum top 25% as candidates, ordered by momentum, pairwise correlation <0.9" (11 ETFs, 7.9% ann.)
- **Flow-augmented momentum (balanced)**: "large+extra-large order net buy top 25% with buy share not overheated (≤75th pct), plus 120d momentum top 25%" (4, 17.1%)
- **Vol-controlled momentum (defense)**: "120d momentum top 25% and short-term vol mean elevated vs long-term (short−long > 0)" (currently 0 holdings = empty position, 16.6%)
Range performance, NAV, dual-caliber holdings (code | name | weight % | price | AUM | tracked index).

---

### 2.6 Module: FI+ Strategies (`/strategy`)

#### 2.6.1 /strategy/income-equity — Dividend-enhanced equity vs CSI Dividend Total Return (H00922.CSI); "monthly rebalance, 0.1% two-way cost." Card (40 stocks, 17.0% ann., +7.6% ann. excess). Range table with excess column; yearly table 2010–2026 with excess; NAV chart; dual-caliber holdings with **stock funnel** "all-share 5,035 → ROE-stability filter 4,469 → dividend-quality Top50 50 → momentum/vol filter 40"; columns: code | name | SW L1 | weight % | dividend yield % ("dividend yield = 1 / (price ÷ per-share dividend), daily ffilled").

#### 2.6.2 /strategy/multi-asset — stock = CSI 800 total return (H00906.CSI), bond = ChinaBond AAA bank tier-2 capital bond wealth (CBA26301.CS); monthly rebalance at month-end, zero cost, close execution; metrics from 2017 (base 2016-12-31 = 100).
- **Table_StrategyOverview** (44+ rows, sortable & filterable): filters (start year 2017–2025; class fixed-proportional / risk-driven / return-driven / insured; thresholds ann.return ≥ / Sharpe ≥ / Calmar ≥ / maxDD ≤). Columns: class | strategy | param | ann.return | ann.vol | maxDD | Sharpe | Calmar | avg equity weight | direction hit-rate | add-hit | reduce-hit | monthly timing gain. Strategy families: constant mix (5/95…30/70); risk budget (10–70%), risk parity, target vol (2–5%), CVaR (90/95/99%); constrained MVO (λ=10/100/1000), momentum-vol MVS (10–50%); CPPI (m=0.5/1/2/3), TIPP (m=0.5/1/2/3), OBPI (3–10%); plus two asset-leg rows. Metric definitions verbatim: "maxDD is magnitude; avg weight = mean month-end equity weight; hit-rate = share of rebalances whose direction matched next-month equity move; timing gain/month = Σ sign(Δw)×next-month return ÷ #rebalances."
- **Section_StrategyDetail**: class tabs → strategy tabs → parameter selector; each shows rule card + formula, 5 KPIs, portfolio NAV vs bond NAV (with optional benchmark combobox of the other 9 strategies / CSI800), monthly equity-weight chart (last point = simulated rebalance), drawdown chart, yearly table incl. cumulative rows.

#### 2.6.3 /about — reverse-chronological changelog (2026-10-20 → present); first-party documentation of feature methodologies.

---

## 3. Underlying Data Model & Schema (reverse-designed)

### 3.1 Data source taxonomy

| Data class | Real-world source |
|---|---|
| CB/EB/stock quotes & static terms | Tushare / Wind / exchange disclosures (field naming `ts_code/conv_price/…` matches Tushare `cb_basic`; ratings and clause texts from prospectuses) |
| Index quotes | CSI (CSI CB 000832, CSI800 TR H00906, CSI Dividend TR H00922), ChinaBond (CBA26301, CBA20921) |
| Exchange holder structure | SSE/SZSE monthly CB investor holding disclosures |
| ETF shares/AUM | Exchange ETF share disclosures (change-date records) + fund companies |
| Fund quarterly holdings | Fund periodic reports (stock/CB/instrument mix, leverage, duration, institutional share) |
| Fund daily NAVs | Daily NAV vendors (fund companies / aggregators) |
| Pension product NAVs | MOHRSS pension product disclosures |
| CSI valuations | CSI index valuation service (EB & preferred stock valuations) |
| Derived analytics (in-house) | prem100 regression, delisting-adjusted BS greeks, 5-factor attribution, triangular chip distribution, kfac HF positions, Kalman durations, RBSA, strategy backtests |

### 3.2 Full static-data file inventory (frontend contract layer)

```
/data/daily_brief.json
/data/cb_market.json / cb_idx.json / cb_supply.json / cb_greeks.json / cb_etf.json
/data/cb_reports/{index.json, explain.json, YYYY-MM.json}
/data/cb_segment_idx.json / cb_segment_idx_style.json / cb_segment_hold.json
/data/clause_board.json / clause_events.json
/data/chips_index.json / chips_market.json / chips/{code}.json / chips/fund_hold.json
/data/bonds/{code}.json / bonds_index.json        (board's 1,095-row basic dataset is inlined in the CbBoard chunk)
/data/eb/{eb_board, eb_market, eb_idx}.json / eb/detail/{code}.json
/data/ps/{ps_board, ps_market}.json
/data/funds_index.json / fund_shelf.json / fip_market.json
/data/fip_strategy/{nav_index, index_meta, enh_nav, enh_meta, holdings}.json
/data/pb_index.json
/data/pb_duration/{meta, group_daily, vs_disclosed}.json
/data/pb_strategy/{shelf, nav_index, index_meta, strategy_nav, strategy_meta, holdings}.json
/data/bond_fund_board.json
/data/pension/{idx, idx_meta, rbsa, manager, scale}.json
/data/etf_index.json / etf_scale.json
/data/pb_etf_index.json / pb_etf_market.json
/data/stk_etf_strategy/{metrics, nav, holdings}.json
/data/cb_strategy/{metrics, nav, holdings}.json
/data/div_strategy/{metrics, nav, holdings}.json
/data/alloc_strategy/strategies.json
```

### 3.3 Core TypeScript schemas (condensed from sections above)

```ts
interface BondDim { ts_code: string; bond_short_name: string; cb_type: "CB"|"EB";
  mkt?: "公募"|"私募"; stk_code: string; stk_short_name: string; exchange: "SH"|"SZ"; board?: string;
  value_date: string; maturity_date: string; list_date: string; delist_date: string|null;
  conv_start_date: string; conv_end_date: string; conv_stop_date: string|null;
  issue_size: number; remain_size: number; par: number; issue_price: number; maturity: number;
  coupon_rate: number; add_rate: number; maturity_call_price: number; pay_per_year: number;
  first_conv_price: number; conv_price: number;
  call_ratio: number; reset_ratio: number; put_ratio: number;
  call_trig_price: number|null; reset_trig_price: number|null; put_trig_price: number|null;
  call_start: string|null; call_end: string|null; put_start: string|null; put_end: string|null;
  call_days: number; call_max_days: number; put_days: number; put_max_days: number;
  reset_days: number; reset_max_days: number;
  issue_rating: string|null; newest_rating: string|null; rating_comp: string|null;
  guarantor: string|null; guarantee_type: string|null; rate_type: string|null;
  rate_clause: string; call_clause: string; put_clause: string; reset_clause: string; conv_clause: string; }

interface CbMarketDailyFact { date: string; n: number; bal: number; mv: number;   // see §2.1.2 for all 28 columns
  conv_v: [number,number,number]; bond_v: [number,number,number]; ptm: [number,number,number];
  ytm: [number,number,number]; conv_prem: [number,number,number]; bond_prem: [number,number,number];
  prem100: number; prem100_n: number; abn_n: number; abn_pct: number; turnover: [number,number,number]; }

interface BondDailyFact { d: string; strbv: number|null; strbp: number|null; convv: number|null;
  convpr: number|null; iv: number|null; convp: number|null; rt: string|null; osz: number|null; dil: number|null; }

interface GreeksFact { d: string; median: number; mean: number; mv: number; n: number;
  iv?: number; vol_his?: number; d_term?: number; g_term?: number; t_term?: number; }

interface AttributionFact { m: string; n: number; ret_actual: number;
  delta: number; gamma: number; theta: number; vega: number; rho: number; residual: number; }

interface ClauseFact { code: string; name: string; stk_name: string; cb_close: number;
  convv: number; conv_price: number; ptm: number; ratio: number; window: number; need: number;
  cnt: number|null; gap: number|null; status: string; exempt_end: string|null;
  last_event: string|null; last_reset?: {date: string; price: number}; note: string|null; days_to?: number|null; }

interface StrategyMetrics { 策略: string; style: string; rebalance: string; desc: string; 起始日期: string;
  [window: string]: { 收益: number; 最大回撤: number; 波动率: number }; }

interface HoldingRow { 代码: string; 名称: string; 权重: number; 价格?: number;
  转股溢价率?: number; 评级?: string; 行业?: string; 规模?: number; 跟踪指数?: string; }
```

### 3.4 Relational DDL (rebuild-ready, PostgreSQL)

```sql
CREATE TABLE dim_bond (
  ts_code VARCHAR(12) PRIMARY KEY, bond_short_name VARCHAR(40) NOT NULL,
  cb_type CHAR(2) NOT NULL, mkt VARCHAR(4), stk_code VARCHAR(12), stk_short_name VARCHAR(40),
  exchange CHAR(2), board VARCHAR(8),
  value_date DATE, maturity_date DATE, list_date DATE, delist_date DATE,
  conv_start_date DATE, conv_end_date DATE, conv_stop_date DATE,
  issue_size NUMERIC(14,2), remain_size NUMERIC(14,2),
  par NUMERIC(8,2), issue_price NUMERIC(8,2), maturity NUMERIC(4,1),
  coupon_rate NUMERIC(5,2), add_rate NUMERIC(5,2), maturity_call_price NUMERIC(6,2), pay_per_year SMALLINT,
  first_conv_price NUMERIC(10,4), conv_price NUMERIC(10,4),
  call_ratio SMALLINT, reset_ratio SMALLINT, put_ratio SMALLINT,
  issue_rating VARCHAR(8), newest_rating VARCHAR(8), rating_comp VARCHAR(60),
  guarantor VARCHAR(60), guarantee_type VARCHAR(20),
  rate_clause TEXT, call_clause TEXT, put_clause TEXT, reset_clause TEXT, conv_clause TEXT, updated DATE);
CREATE INDEX idx_bond_type ON dim_bond (cb_type, delist_date);

CREATE TABLE fact_bond_daily (
  ts_code VARCHAR(12), trade_date DATE,
  close NUMERIC(9,3), conv_v NUMERIC(9,3), bond_v NUMERIC(9,3),
  conv_prem NUMERIC(7,3), bond_prem NUMERIC(7,3), ytm NUMERIC(7,3),
  iv NUMERIC(7,3), delta NUMERIC(6,4), turnover NUMERIC(7,3),
  amount NUMERIC(14,3), remain_size NUMERIC(14,2), rating VARCHAR(8),
  PRIMARY KEY (ts_code, trade_date));
CREATE INDEX idx_bd_date ON fact_bond_daily (trade_date);

CREATE TABLE fact_cb_market_daily (     -- the 28-column panel
  trade_date DATE PRIMARY KEY,
  n SMALLINT, bal NUMERIC(10,1), mv NUMERIC(10,1),
  conv_v_med NUMERIC(8,2), conv_v_avg NUMERIC(8,2), conv_v_mv NUMERIC(8,2),
  bond_v_med NUMERIC(8,2), bond_v_avg NUMERIC(8,2), bond_v_mv NUMERIC(8,2),
  ptm_med NUMERIC(5,2), ptm_avg NUMERIC(5,2), ptm_mv NUMERIC(5,2),
  ytm_med NUMERIC(7,2), ytm_avg NUMERIC(7,2), ytm_mv NUMERIC(7,2),
  conv_prem_med NUMERIC(7,2), conv_prem_avg NUMERIC(7,2), conv_prem_mv NUMERIC(7,2),
  bond_prem_med NUMERIC(7,2), bond_prem_avg NUMERIC(7,2), bond_prem_mv NUMERIC(7,2),
  prem100 NUMERIC(6,2), prem100_n SMALLINT, abn_n SMALLINT, abn_pct NUMERIC(5,2),
  turn_med NUMERIC(6,2), turn_avg NUMERIC(6,2), turn_mv NUMERIC(6,2));

CREATE TABLE fact_cb_greeks_daily (
  trade_date DATE PRIMARY KEY,
  delta_med NUMERIC(6,3), delta_mean NUMERIC(6,3), delta_mv NUMERIC(6,3), delta_n SMALLINT,
  exp_med NUMERIC(7,3), exp_mean NUMERIC(7,3), exp_mv NUMERIC(7,3), exp_n SMALLINT,
  iv_med NUMERIC(6,2), vol_his_med NUMERIC(6,2),
  d_term NUMERIC(6,2), g_term NUMERIC(6,2), t_term NUMERIC(6,2),
  above_n SMALLINT, above_pct NUMERIC(5,1));

CREATE TABLE fact_cb_attribution_monthly (
  month CHAR(7) PRIMARY KEY, n SMALLINT, ret_actual NUMERIC(7,3),
  delta NUMERIC(7,3), gamma NUMERIC(7,3), theta NUMERIC(7,3),
  vega NUMERIC(7,3), rho NUMERIC(7,3), residual NUMERIC(7,3));

CREATE TABLE fact_clause_event (
  id BIGSERIAL PRIMARY KEY, ts_code VARCHAR(12), event_date DATE,
  event_type VARCHAR(8), executed BOOLEAN, detail TEXT,
  ret_5d NUMERIC(7,3), ret_10d NUMERIC(7,3), ret_20d NUMERIC(7,3),
  excess_5d NUMERIC(7,3), excess_10d NUMERIC(7,3), excess_20d NUMERIC(7,3));

CREATE TABLE fact_cb_supply_monthly (
  month CHAR(7) PRIMARY KEY, issue_n SMALLINT, issue_sz NUMERIC(9,1),
  conv_n SMALLINT, conv_sz NUMERIC(9,1), other_n SMALLINT, other_sz NUMERIC(9,1),
  dig_conv NUMERIC(9,1), dig_cash NUMERIC(9,1));

CREATE TABLE fact_fund_hf_position (
  fund_code VARCHAR(12), trade_date DATE,
  stock_pos NUMERIC(6,2), cb_pos NUMERIC(6,2),           -- kfac estimates (%)
  stock_pos_disc NUMERIC(6,2), cb_pos_disc NUMERIC(6,2), -- quarterly disclosures
  PRIMARY KEY (fund_code, trade_date));

CREATE TABLE fact_fund_duration (
  fund_code VARCHAR(12), trade_date DATE, dur_kalman NUMERIC(5,2),
  group_key VARCHAR(24),                                 -- category × instrument style
  PRIMARY KEY (fund_code, trade_date));
```

---

## 4. API Contracts for Reconstruction

### 4.1 Dynamic endpoints (auth only)

**POST /api/auth/login**
```json
// Request
{ "username": "evian", "password": "••••" }
// 200
{ "ok": true, "user": { "username": "evian", "displayName": "EvianZhou" } }
// 401
{ "ok": false, "error": "用户名或密码错误" }
```

**GET /api/auth/me** (cookie)
```json
{ "authenticated": true, "username": "evian", "displayName": "EvianZhou" }
// 401 → redirect to login
```

**POST /api/auth/register**
```
{ "name": "…", "username": "…", "email": "<corporate email>", "password": "≥8 chars, uppercase + digit" }
```

### 4.2 Static data endpoints — response samples

**GET /data/cb_market.json**
```json
{ "updated": "2026-10-29",
  "daily": { "cal": ["20041019", "…", "20261029"],
             "rows": [[319, 4737.7, 5967.95, 81.32, 91.43, 94.79, 106.55, 105.57, 106.6,
                        2.69, 3.0, 2.59, -3.53, -9.65, -9.06, 59.34, 73.55, 63.56,
                        19.83, 36.37, 30.46, 41.78, 293, 6, 1.88, 4.21, 16.14, 8.64]] },
  "holder": { "sse": { "months": {"0": "2016-07"}, "investors": {"0": "公募基金"},
                       "hold": {"0": {"0": 125.74}}, "trade": {"0": {"0": 67.33}},
                       "holdUnit": "面值(亿元)", "tradeUnit": "成交金额(亿元)" },
              "szse": { "…": "…" } } }
```
For cb_greeks.json / cb_etf.json / cb_segment_hold.json / clause_board.json / eb_board.json / ps_board.json / cb_strategy/*.json see §2.1–2.6 samples.

### 4.3 Frontend rendering contract (must-haves for rebuild)

1. **Compact arrays**: time series use column/row arrays, not object arrays (`[date, v1..vk]` rows). Sizes: cb_market 449KB, cb_idx 208KB, board dataset inlined 757KB chunk.
2. **Client-side window math**: ChartRangePicker changes re-base NAVs and recompute cumulative/annualized/vol/maxDD/Sharpe/Calmar in the browser; annualization = 243 trading days.
3. **Dual-caliber holdings**: every strategy page renders Actual holdings (last official rebalance) and Simulated rebalance (latest trade date), with date & count captions.
4. **Legend-click toggling** (ECharts default); right axis typically count / cumulative / share.
5. **Tables**: sortable headers; nulls "—"; code cells deep-link to detail pages; 50/page default.
6. **Downloads**: per-chart CSV sliced to the active window.
7. **Monthly review / research notes**: index.json-driven combobox + lazy `{YYYY-MM}.json` (title + long-form body).

---

## Appendix A: Verbatim formula quick reference

| Metric | Formula / methodology |
|---|---|
| Par-100 premium | daily cross-section regression y = a·e^(−b·x) + c (y = conversion premium, x = parity/100), value at x=1; sample = no announced redemption & maturity >0.5y |
| Turnover | daily amount / total market cap; cap-weighted by bond cap |
| Abnormal bond | price ≥130 AND conversion premium ≥30% AND turnover ≥100% |
| CB theoretical value | V = survival_prob × [ s·N(d1) + X·e^(−r·t)·(1−N(d2)) ], r = floor YTM, delisting line = max(1/price, ¥300mn/cap) × parity |
| Delta | ∂V/∂s = N(d1)·survival_prob + delisting correction |
| Expected return (ann.) | ( Δ·s·r_f + ½·Γ·(s·σ_his)² + Θ ) / P, r_f = 4% |
| Above-floor share | #(E[ret] > floor YTM) / valid sample |
| Monthly attribution | return = Δ+Γ+Θ+vega+ρ+residual via full revaluation (s→σ→t→r sequential re-pricing) |
| ETF AUM | shares × close; shares recorded on change dates, forward-filled |
| Net subscription | Δ shares × close |
| CB/EB indices | monthly rebalance base 1000; coupon-adjusted; 243-day annualization; balance > ¥30mn; exclude triggered-redemption/delisted |
| FI+ vol tier | 4-quarter mean (stock + CB×0.5): <10 low / 10–20 mid / ≥20 high; CB mean >40% → CB fund |
| kfac HF position | Kalman regression + holdings mark-to-market, anchored to quarterly disclosures; MAE ≤3pp good |
| Kalman duration | unconstrained TVC regression (burn=60, q_beta=1e-4) on 930865 + 931080 |
| Instrument style | rolling-2Q instrument share >60% (priority rate→credit→cash-like; else mixed) |
| Composite position (FOF) | latest 2 disclosures (stock + 0.5×CB) / NAV |
| Timing gain/month | Σ sign(Δw) × next-month return ÷ #rebalances |
| 3-D ability | offense = up-window de-positioned win-rate + elasticity; defense = no-loss ratio in down windows; stability = CV ranks of 60d rolling betas vs CSI800 & CBA20921 |
| Dividend yield (strategy) | 1 ÷ (price ÷ per-share dividend), daily forward-filled |

## Appendix B: Rebuild acceptance checklist

- [ ] All 7 top modules + 27 sub-routes navigable (back-links and secondary tabs included)
- [ ] Auth wall: any route unauthenticated → `/` login page (login/register tabs)
- [ ] Every chart: time-window button group (All/5Y/3Y/1Y/YTD/custom) + CSV download
- [ ] Dual-caliber holdings tables on all 5 strategy pages
- [ ] Index performance quartet: cumulative / annualized / ann.vol / maxDD
- [ ] Dark/light toggle with chart palette sync
- [ ] All /data/*.json produced per §3.2 with `updated` matching the on-page "data as of" caption
