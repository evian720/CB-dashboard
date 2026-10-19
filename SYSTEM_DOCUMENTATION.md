# CB Momentum Study — System Documentation (LLM Rebuild Reference)

> Purpose: a complete, self-contained specification of the convertible-bond (CB 可转债)
> research system, precise enough for an LLM to re-implement it — more robustly.
> Chinese identifiers (column names, statuses) are **load-bearing**; keep them verbatim.

---

## 1. System Overview

A local, offline-first research platform for Chinese exchange-traded **public convertible
bonds** (公募可转债), built on Streamlit + pandas + parquet, with:

| Layer | Components |
|---|---|
| **Data** | Tushare Pro (primary) → WindPy (fallback) → jisilu.cn (redemption) → cninfo (announcements). Local parquet store under `data/`. |
| **Factors** | `cb_factors.py` (CICC-report-derived CB factors), `stock_factors.py` (stock-side factors: capital flow, chips, industry momentum, volatility). |
| **Backtest** | `backtest.py`: per-trade engine + portfolio engine, both no-look-ahead, with costs. `strategy_analysis.py`: attribution/MC/OOS/walk-forward. |
| **Strategy factory** | `strategy_llm.py` (NL→code via GLM), `factor_templates.py` (B-track), `brute_force.py` (D-track), `batch_runner.py` (headless batches), `experiment_db.py` (SQLite). |
| **Virtual hedge fund** | `virtual_fund/`: 13 AI PM personas + CIO + risk engine + self-evolving playbooks; replay (batch) & live (daily) modes; background runner. |
| **UI** | `dashboard.py` + 14 `page_*.py` modules; bilingual (zh/en) via `i18n.py`; corporate-dashboard theme w/ dark mode via `ui_style.py`. |
| **Ops** | `update_data.py` (one-button update), `run_checks.py` (67-test regression), `page_status.py`/`page_settings.py`. |

Key entry points:
- `启动Dashboard.bat` → `_launch.py` → `streamlit run dashboard.py --server.port 8501`
- One-button update: sidebar 更新数据 → `update_data.update_all(progress_cb)`
- Regression: `python run_checks.py [--fast|--with-llm]`

---

## 2. Data Dictionary (all files under `data/`)

> **Unit conventions (CRITICAL — historically buggy):**
> - 成交额 (amount) is **元 (CNY)** in every price file.
> - CB 成交量 unit = 手 (1 手 = 10 张 = 1000 CNY face) ⇒ implied price 成交额/成交量 ≈ price×10.
> - Stock prices are **pure 后复权 (hfq)** = raw close × cumulative `adj_factor` (stable base, append-safe).
> - ETF 成交额 in 元, 成交量 in 手.
> - 主力净流入/小单净流入 in 元. 转股溢价率 in %. 剩余规模 in 亿元.
> - **Never mix adjustment bases or amount units in one series** — see §3 invariants.

### 2.1 Price / market data

| File | Columns | Source | Update |
|---|---|---|---|
| `cb_prices/cb_daily_ohlc.parquet` (+.csv) | `date, 转债代码, 开盘价, 最高价, 最低价, 收盘价, 成交量(手), 成交额(元), 换手率(%), 转股溢价率(%)` | Tushare `cb_daily` (full rebuild 2025-01-02→now; incl. bonds delisted mid-period ⇒ less survivorship bias). 换手率 computed = 成交量×10/(剩余规模亿×1e6)×100. | date-batch incremental `_ts_update_cb_prices` |
| `stock_prices/stock_daily_ohlc.parquet` | `date, 正股代码, 开盘价, 最高价, 最低价, 收盘价, 成交量, 成交额(元)` — all 4 prices × cumulative adj_factor (pure hfq) | Tushare `daily` + `adj_factor` (trade-date batch) | incremental ×adj_factor (never raw!) |
| `etf_prices/etf_daily_ohlc.parquet` | `date, ETF代码, 开盘价…收盘价, 成交量, 成交额(元)` — raw (no adjustment) | Tushare `fund_daily` (511380.SH 可转债ETF) | incremental via `fetch_fund_daily` |
| `cb_prices_delisted/cb_daily_ohlc.parquet` | same as cb_prices | **Wind-only** (Tushare lacks premium & full coverage for delisted CBs). Internal pure-Wind, units fixed to 元. | manual `download_cb_delisted.py` |
| `stock_prices_delisted/stock_daily_ohlc.parquet` | same as stock_prices | Wind `PriceAdj=B`. Base verified ≡ Tushare hfq (261k overlap rows, ratio 1.000±0.0004). | manual |
| `delisted_info.parquet` | `转债代码, 转债简称, 正股代码, 正股简称, 起息日, 到期日, 退市日` | Wind | manual |

### 2.2 Reference / snapshot data

| File | Columns | Source |
|---|---|---|
| `basic_info.parquet` | `转债代码, 转债简称, 正股代码, 正股简称, 起息日, 到期日, 退市日, 票面利率, 发行价, 剩余规模(亿), 最新转股价, 最新评级` | Tushare `cb_basic` (full replace; incl. delisted w/ 退市日; public-filtered) |
| `cb_issuing.parquet` | `正股代码…公告日期, 方案进度, 距今天数, 发行规模(亿), 期限(年), 6 stage dates + days…` | **Wind-only** `wset convertiblebondissuingpreplan22` |
| `cb_redeem.parquet` | see `fetch_cb_redeem.normalize_redeem` (转债代码/强赎进度/距满足天数/剩余规模(亿)/状态/抓取时间…) | jisilu webapi `https://www.jisilu.cn/webapi/cb/redeem/` |
| `monitor_snapshot.parquet` | `转债代码, sec_name, 正股代码, 评级, 正股总市值, 抓取时间` | **Wind-only** snapshot |
| `cb_announcements.parquet` | `公告时间, 证券代码, 证券简称, 公告标题, 公告类型, PDF链接` | cninfo POST API `category_kzzq_szsh` |

### 2.3 Factor-source data (all pure Tushare)

| File | Columns | Notes |
|---|---|---|
| `fund_flow.parquet` | `date, 正股代码, 主力净流入(元), 小单净流入(元)` | 主力=(buy_lg+buy_elg−sell_lg−sell_elg)×1e4; 小单=(buy_sm−sell_sm)×1e4. Full history 2025-01→. |
| `sw_index.parquet` | `date, 指数代码, 指数名称, 收盘价` | 31 SW L1 industries via `sw_daily` (**`.SI` suffix**; `index_daily_sw` does NOT exist at this token level). |
| `sector_map.parquet` | `正股代码, 申万行业` | `index_classify`(L1, SW2021) + `index_member_all` (`ts_code` column!). |
| `concept_map.parquet` | `正股代码, 热门概念` | `concept` + `concept_detail`, `;`-joined top8. |
| `chip_distribution.parquet` | `date, 正股代码, 成本5分位, 成本50分位, 成本95分位, 加权成本, 获利盘比例(%)` | `cyq_perf` — **must keep ALL 5/50/95 columns on incremental merge** (bug fixed once). |
| `cb_holders.parquet` | `转债代码, 报告期, 排名, 持有人, 持有数量, 持仓占比` | `top10_cb_holders` — report period column is **`end_date`** (NOT ann_date); ~139 disclosed bonds only; quarterly. |
| `stock_vol.parquet` | `date, 正股代码, vol3, vol5, vol10, vol30, vol60 (annualized %), vol_ratio(=vol3/vol10−1), vol3_up(连升天数)` | derived locally by `stock_factors.refresh_stock_vol()` after every price update; suspension-gap-masked returns (calendar gap >15d ⇒ return=NaN). |

### 2.4 State / outputs

| File | Purpose |
|---|---|
| `config.json` | `zhipu_api_key, zhipu_base_url, zhipu_model, tushare_token` |
| `signal_state.json`(+.bak) | live strategy positions (page_signals) |
| `watchlist.json` | watchlist + alerts |
| `ui_prefs.json` | `{"dark": bool}` |
| `monitor_style.json` | monitor-table styling rules |
| `virtual_fund/live_state.json` | live fund: pods, playbooks (incl. tunables), executed_log, auto_mode |
| `virtual_fund/bg_status.json` | background replay runner status (heartbeat `updated`; stale>180s ⇒ dead) |
| `virtual_fund/<session_id>/` | replay sessions: `meta.json(complete:bool), nav_hist.json, pods.json, orders.jsonl, risk_events.jsonl, ic_log.jsonl, playbooks.json`. Checkpointed **daily**; interrupted runs stay visible (⚠️未完成). `session_id` = `%Y%m%d_%H%M%S` + uuid4 hex[:4]. |
| `backtest_history/*.json(+_trades.csv)` | per-run archives from page_backtest |
| `experiments.db` | SQLite (WAL) strategy-lab results, table `runs` (24 cols, unique `(batch_id, description)`) |
| `batch_status/*.json` | batch progress files polled by page_lab |

---

## 3. Hard Invariants (lessons — a rebuild MUST preserve these)

1. **Pure-hfq stock base**: stock prices = raw × cumulative adj_factor (Tushare), never
   "normalized to latest" — normalized bases drift, breaking appended increments
   (2026-09-07 incident: 186 fake −50%~−97% single-day returns across the seam).
2. **Amount units are 元 everywhere**. Wind wrote 千元/万元 for CB/ETF amounts — the
   2026-09-07 unit seam corrupted 850k rows. Guard tests: `tests/test_data_seams.py`
   (monthly implied-price 成交额/成交量 must stay within [0.2×, 5×] of global median;
   no single day with >20 bonds moving >30%(CB)/50%(stock)).
3. **Suspension gaps are not daily returns**: when computing stock returns/vol, mask
   returns whose calendar gap >15 days (e.g. 300630.SZ resumed after 1yr suspension:
   −74% fake "daily" return).
4. **Provider selection must compare exact values** (`use_mock == "mock"`), never
   truthiness of a radio string (bug: every "real-data replay" silently ran mock).
5. **Persistence round-trips**: `pods_from_json` returns bare PodState dicts — must be
   re-wrapped into Pod objects; it accepts a JSON **string** (a dict input raised
   TypeError swallowed by except ⇒ live state silently reset on every reload).
6. **Adding PMs must backfill pods** on old live_state (missing-key KeyError) and must
   update all 4 regime tables.
7. **Google Drive filesystem**: all JSON writes atomic (tmp + os.replace, `io_utils`);
   status-file writes may transiently fail — guards must use in-process memory, not files.
8. **Long jobs never run in the Streamlit script thread** (browser refresh/interaction
   kills it). Replay runs in a daemon thread (`virtual_fund/bg_run.py`) with daily
   checkpoints; only a server restart kills it.
9. **LLM calls need timeout (150s) + max_retries=1**; on failure the caller falls back
   to deterministic heuristics — LLM can slow but never break the system.
10. **No look-ahead anywhere**: signals at t close → execution t+1 open (or later);
    `strategy_llm.test_signal` enforces via two-direction canary; panels' derived
    columns use right-aligned rolling / positive shifts only.
11. **Public-CB universe** (`universe.py`): prefix whitelist
    `{110/111/113/118.SH, 123/127/128.SZ}` + name-exclude regex `定转|定\d|EB\d?|EB$`.
    Applied at every load layer.
12. **Monitor "in-market" filter**: Tushare files only contain traded days ⇒ restrict
    to bonds with data on the file's max date (else delisted bonds leak in via tail(2)).
13. **st.tabs selection resets on rerun** — use a session_state-backed radio as view
    switcher instead (or accept resets).
14. **Streamlit duplicate widget keys** crash pages — every widget in a multi-render
    path needs a unique `key=`.

---

## 4. Module Reference

### 4.1 Data ingestion

**`tushare_client.py`** — Tushare Pro wrapper. Token mgmt (`load_token/save_token/
has_token/get_pro/test_token`), `_retry()` rate-limit handler (~0.15s sleep).
`TsClient` methods: `fetch_cb_basic()` (full, no exchange filter — <2000 rows),
`fetch_cb_daily(codes=None,start,end)` (per-code OR per-trade-date batch; converts via
`ts_to_cb_daily`: amount 万元×1e4→元), `fetch_stock_daily`, `fetch_adj_factor`,
`fetch_stock_daily_hfq` (×cumulative adj_factor), `fetch_fund_daily` (amount 千元×1e3→元),
`fetch_stock_turnover` (daily_basic), `fetch_trade_cal`, `fetch_sw_industry_map`
(index_classify L1 SW2021 + index_member_all), `fetch_moneyflow` (主力+小单净流入, 元),
`fetch_sw_index` (**sw_daily**, .SI codes), `fetch_chip_distribution` (cyq_perf),
`fetch_top10_holders` (end_date→最新报告期 filter, Chinese columns).

**`update_data.py`** — one-button pipeline `update_all(progress_cb) -> (success, messages)`:
强赎(jisilu) → Tushare-available check → per-step **Tushare-first with Wind fallback**
(basic, cb prices, stock prices×adj, ETF, sector=moneyflow/sw_index/chip/holders) →
Wind-only steps (issuing, monitor snapshot) gated on Wind connectivity (catch **OSError**
on isconnected — Wind crashes with access violation) → `stock_factors.refresh_stock_vol()`
→ `check_data_quality()` gate (coverage %, date alignment). Incremental CB/stock/moneyflow
use **trade-date batch** (1 call/day), chip last 5 days, holders 12h skip. Helpers
`_ts_update_*` are the Tushare paths; `update_*` are the legacy Wind paths.

**`download_ts.py`** — full re-download CLI: `--all/--prices/--basic/--sector/--chip --start`.
**`download_cb_data.py` / `download_sector_data.py` / `download_cb_delisted.py` /
`download_cb_issuing.py`** — legacy Wind downloaders (see §2 for outputs). Delisted files
remain Wind-sourced (Tushare coverage incomplete: no cb_over_rate, ~1.9% price diffs).

**`fetch_cb_redeem.py`** — jisilu redeem API (Vue-SPA backend JSON; home-page cookie warm-up,
UA/Referer/X-Requested-With headers). `normalize_redeem` → 21 Chinese columns incl.
状态 (强赎中/已公告强赎/已满足强赎条件/不强赎/接近触发) & 剩余规模(亿).
**`fetch_cb_announcements.py`** — cninfo paginated POST; `to_trade_date` maps post-15:00
announcements to next trading day via calendar from cb_prices.

### 4.2 Factors

**`cb_factors.py`** (CICC-report factors, panel/cross-section):
`fit_premium_curve(parity,premium)` → (a,b,c,r²) for `premium=a·exp(−b·x)+c`, x=parity/100
(reject <30 pts or r²<0.15); `curve_value_at`; `premium_residuals(cs)` → `prem_resid`
(negative=cheap vs curve; NaN on low-R² days); `sweet_rate(cs, price_max=115, prem_max=25)`;
`add_delta_emp(panel, window=60)` (rolling cb~stock beta, clip 0–1.5);
`add_chip_pnl(panel, window=250)` (close/amount-weighted-VWAP−1);
`weak_perf(cs_hist, day_idx)`; `momentum_crowding(...)` (pairwise corr of top-momentum
stocks, ≥0.4 = crowded); `parity_bucket_premiums(cs)` (low/mid/high medians + inversion flag);
`mkt_trend(cs_hist, day_idx)` (equal-weight index vs MA20/MA55); `comp_obj_signal(cs_prev, cs)`
(上涨缩溢价 top / 下跌拉溢价 bottom).

**`stock_factors.py`** (stock-side; merges onto panels/cross-sections):
`STOCK_FACTOR_COLS = [mflow5, sm_out5, stock_winner, chip_conc, ind_mom20, inst_hold,
sw_industry, vol3, vol_ratio, vol3_up]`.
- `compute_stock_vol(...)` → per (正股代码,date): vol{3,5,10,30,60} annualized % +
  vol_ratio + vol3_up (consecutive-up days); masks suspension gaps (>15 calendar days).
- `refresh_stock_vol()` — full recompute to `data/stock_vol.parquet` (runs in update_all).
- `add_stock_factors(big)` — merges fund_flow (5d rolling sums ÷ 5d stock_amount →
  mflow5/sm_out5), chip (获利盘→stock_winner, (95pct−5pct)/加权成本→chip_conc),
  sector_map+sw_index (20d industry momentum → ind_mom20, name → sw_industry),
  cb_holders (institutional share via regex 基金|保险|社保|养老|年金|券商|理财|信托|资管|
  工银|中银|… → inst_hold). Missing sources ⇒ whole columns NaN (screens self-filter).

### 4.3 Backtest engine — `backtest.py`

- `load_aligned_panels(cb_code=None, use_cache=True, include_delisted=False)` →
  `{cb_code: DataFrame[date, cb_open/high/low/close/volume/amount/turn/premium,
  stock_open/high/low/close/volume/amount, stock_ret]}`, inner-joined on date, public-filtered,
  double-slot cache returning **copies**; `include_delisted=True` merges the two delisted
  parquets + mapping (overlap: delisted row wins).
- `_add_market_context(panels)` injects per-day `mkt_avg/med_premium, mkt_avg/med_price,
  mkt_n_bonds, mkt_pct_rank_{premium,price}, mkt_sweet_rate` + per-bond `delta_emp,
  chip_pnl, prem_resid` (idempotent, fingerprint-cached).
- `prepare_signals(signal_fn, …, rank_fn, top_n, market_context)` — shared signal step.
- `run_backtest(...)` — per-trade engine: every signal fills at per_trade notional;
  `buy_rule next_open|next_close`, `sell_mode fixed_days|stop_loss|take_profit|
  trailing`, costs `commission=2bp both sides, slippage=10bp against you`;
  suspension-aware sell-day fallback. Returns `(trades_df, stats_dict_zh_keys, benchmark_df)`
  (stats keys: 交易笔数/胜率/总盈亏/累计收益率(每笔独立)/最大回撤率/夏普/Calmar/盈亏比…).
- `run_portfolio_backtest(...)` — finite capital, max_positions slots, same-day sell
  proceeds NOT reusable at open (anti-optimistic), union-calendar equity, rejection
  statuses 未成交(仓位满/已持仓/资金不足). Returns `(trades_df, stats, equity_df)`.
- `apply_topn(sig_panels, rank_fn, top_n)`; `BUILTIN_RANK_FNS = {oversold, low_premium,
  high_turnover, high_amount}`.
- History: `save_backtest_history/list/delete/clear_all` in `data/backtest_history/`.

**`strategy_analysis.py`** — post-hoc analytics on one run: `run_all_analyses` (MFE/MAE,
holding decay, exit breakdown, price/premium/liquidity buckets, monthly heatmap,
rule-based insights); `random_baseline` (MC null, verdict ≥95pct strong / ≥80 mild);
`sensitivity_sweep` (1-D-at-a-time); `run_ablation` (remove-one-condition);
`build_digest` (compact facts for LLM); `run_oos_comparison` (IS/OOS verdicts);
`run_segment_analysis` (walk-forward consistency).

### 4.4 Strategy factory

**`strategy_llm.py`** — GLM (zhipuai SDK, glm-5.3, timeout 150s, 1 retry; OpenAI-compatible
fallback). `generate_strategy(NL)→code`; sandboxed `compile_signal/compile_rank_fn`
(whitelist imports pandas/numpy/math/datetime/statistics, ~35 safe builtins);
`test_signal` two-direction canary anti-look-ahead; `ablation_variants`;
`generate_idea_descriptions` (C-track); `diagnose_strategy`; `generate_batch_from_diagnosis`.
Prompt contract documents the full panel column list incl. mkt_*/factor columns.

**`factor_templates.py`** (B) — 3-dim deterministic grid (price/premium/momentum) codegen.
**`brute_force.py`** (D) — 11 slots (price, premium, dblow, smom, cmom, turn, amt, mkt,
resid, chip, delta) coarse options; `generate_grid(cap=2000)` deterministic; campaign-based
dedup via `experiment_db.has_run_campaign`.
**`batch_runner.py`** — headless subprocess: spec JSON → per-strategy pipeline (codegen →
canary → both engines → MC percentile → optional OOS → SQLite) with resume, failure
isolation, pollable `data/batch_status/<id>.json`.
**`experiment_db.py`** — SQLite WAL, table `runs` (24 cols), `init_db/has_run/
has_run_campaign/insert_run/query_runs/list_batches/delete_batch`; connections ALWAYS
closed (Windows locks).

### 4.5 Virtual hedge fund — `virtual_fund/`

- `config.py`: `INITIAL_POD_CAPITAL=10M`, commission 万1, slippage base 10bp + small-scale
  penalty, `MAX_POSITION_PCT=0.15`, `REDEEM_REJECT_PRICE=130`, drawdown tiers 2.5%/5%,
  freeze 3d, CIO multiplier clamp [0.5,1.5], `TOP_K_CANDIDATES=5`.
- `schemas.py`: dataclasses TradeProposal/Order/Position/PodState/RiskEvent/ICLog/Reflection;
  `pods_to_json/pods_from_json` (⚠️ returns PodState, caller wraps into Pod).
- `engine.py`: `Pod` (nav/drawdown/HWM/apply_buy/apply_sell, `trading_capital=cash×multiplier`);
  `ExecutionSimulator` (T+0, fee=max(notional×1bp,1), scale-dependent slippage);
  `RiskEngine.pre_trade` (redeem-reject, frozen, 15% concentration, tier gross-limit,
  cash buffer) + `post_trade` (stop-loss, Tier1 trim worst-half, Tier2 force-liquidate+freeze).
- `agents.py`: **13 PM personas** — each `dict(key, style, screen, stop, thesis, tunables)`
  where `screen(cs, k, params)` filters the daily cross-section (CS_COLS below) and
  `tunables={param: [lo, hi, default, strict_dir]}` are **evolvable entry thresholds** stored
  in the playbook. PMs: low_price, balanced, cb_momentum, stock_momentum, mean_rev, event,
  rel_value, chip_rev, sweet_spot (CICC) + mflow_alpha, sector_ride, stock_chip, vol_expand
  (stock-factor). `BasePM.research(cs)` → proposals (heuristic or `_llm_refine` conviction
  grading with playbook in prompt); `reflect(date, day_pnl, pod_dd)` evolves playbook:
  drawdown⇒stop×0.85, conviction+0.03, tunables→strict 12% of range; profit day⇒loosen 4%;
  eventful LLM days (dd≥Tier1 or day_pnl<−1% capital) may write new heuristics + tunable
  adjustments (clipped to bounds). `CIOAgent.assess_regime(cs_hist)` → EQUITY_BULL/
  SPECULATIVE_ROTATION/DEFENSIVE_BEAR/RANGE_BOUND (+sweet rate, MA20/55 gates, crowding,
  parity inversion); `challenge()` → per-PM multipliers + per-proposal accept/cut/drop
  (`REGIME_STYLE_FIT` must list every persona × 4 regimes).
  Cross-section contract `CS_COLS`: cb_code, close, open, premium, parity, turn, amount,
  stock_ret5/20, cb_ret5/20, turn_ma20, cb_max5, redeem_near, scale_yi, name, delta_emp,
  chip_pnl, prem_resid + STOCK_FACTOR_COLS; mkt_ columns for CIO.
- `providers.py`: `LocalPanelProvider` (panels + delta/chip + `add_stock_factors` +
  per-day premium_residuals/sweet_rate, static scale/redeem from jisilu snapshot) and
  `MockProvider` (80 bonds × 150 days synthetic, 3 regimes; stock-factor columns NaN).
- `orchestrator.py`: 5-phase daily loop (研究 4-thread-parallel → IC → 风控+open execution
  → close risk → reflect), NAV history, incremental checkpoint `_persist(complete=False)`
  daily + final complete=True; unique session_id.
- `live.py`: `LiveFund` persistent daily mode — `run_daily_cycle()` = research →
  **auto paper-fill** (`auto_execute`: sells first, buys at close+slippage, size=
  NAV×size_pct) → risk (forced sells auto-executed) → evolve; manual `confirm_fills`
  path retained (auto_mode toggle, persisted); `mark_and_evolve(auto_execute_forced)`;
  PM-backfill on load; playbook tunables self-heal.
- `bg_run.py`: background replay runner — in-memory lock guard (not file!), heartbeat
  status file, stale>180s, unique sessions; UI polls via `st.fragment(run_every=5)`.

### 4.6 UI pages (dashboard.py routes)

`dashboard.py`: sidebar language/dark toggles, nav radio (overview/single/monitor/sector/
issuing/backtest/lab/fund/announcements/redeem/signals/status/settings), global status bar
(`_get_status/_get_quality` cached, 更新数据 button with source label 📡Tushare优先+Wind回退).
Page modules (each exposes `page_*()`):

| Page | Highlights |
|---|---|
| `page_overview` | market KPIs, 100-parity-premium curve (exp fit + R² panel + window selector), distributions, redeem pressure, watchlist+alerts |
| `page_single` | single-bond search (pinyin), price/parity/premium chart |
| `page_monitor` | 18-col cross-section (incl. 波动10/30/60d%), filters, heatmaps, persistent styling (`monitor_style.json`), alive-bond filter |
| `page_sector` | SW L1 rotation: stock-vs-CB perf, lag signals, weekly heatmap, flows |
| `page_issuing` | issuance pipeline stages + per-plan timeline |
| `page_backtest` | NL→LLM code→both engines→5 analysis tabs (Result/Quality/Attribution/Robustness/AI) + history |
| `page_lab` | batch tracks A/B/C/D via batch_runner subprocess; records browser on experiments.db |
| `page_fund` | virtual fund: 实盘模式 (auto-cycle + manual fills + paper-trade log + playbook viewer), 批量运行 (bg runner + live log), 实验记录 |
| `page_announcements` / `page_redeem` / `page_signals` | cninfo announcements; redemption monitor; live signal engine |
| `page_settings` | Tushare token, data-source status, API availability checks |
| `page_status` | storage usage by bucket + file freshness table (source/frequency/status) |

**Infra**: `app_common.py` (cached loaders, 转股价值=收盘价/(1+溢价率/100)), `i18n.py`
(~700 keys, `t(key)` zh-default), `ui_style.py` (CSS-var light/dark, plotly patch,
heatmap stylers), `charts.py` (4 shared figure builders), `universe.py`, `io_utils.py`
(atomic writes), `watchlist.py`, `run_checks.py` (custom runner, 67 tests, --fast/--with-llm).

---

## 5. Key Data Flows

```
Tushare/Wind/jisilu/cninfo
   → update_data.update_all (Tushare-first, Wind fallback, quality gate)
   → data/*.parquet
   → backtest.load_aligned_panels (+market context)  ← strategy_llm/factor_templates/brute_force
   → run_backtest / run_portfolio_backtest             (batch_runner, page_backtest, page_lab)
   → strategy_analysis (MC/OOS/segments) → experiments.db / backtest_history
   → signal_gen (live next-day signals) → page_signals

data/*.parquet
   → stock_factors/cb_factors → virtual_fund.providers cross-sections (CS_COLS)
   → agents (13 PMs, tunable screens, LLM grading) → CIO (regime, multipliers)
   → engine (risk + sim fills) → orchestrator (replay, checkpoints) / live (daily cycle)
   → data/virtual_fund/<session> + live_state.json → page_fund
```

---

## 6. Conventions & Rebuild Guidance

1. **Column names are Chinese** in all raw files; engine-internal renames add cb_/stock_
   prefixes. Keep both layers exactly as documented.
2. **Config**: `config.json` holds both tokens; never commit it.
3. **Every long job**: background thread + heartbeat status file + incremental checkpoints
   + stale detection; UI only polls.
4. **Every LLM boundary**: timeout + retry cap + deterministic fallback; outputs clipped
   to safe ranges; prompts carry the playbook (incl. tunables) for closed-loop learning.
5. **Every data append**: same units/base as history (see §3); guard with
   `tests/test_data_seams.py`-style statistical checks, not just schema checks.
6. **Every persisted state**: atomic write + .bak + version-tolerant load (backfill new
   keys, drop removed ones).
7. **Determinism**: LLM-off paths must be bit-reproducible (seeded, ordered merges,
   no wall-clock in logic) — enforced by tests.
8. **Tests** (`run_checks.py`, 67): data seams, monitor assembly, backtest math hand-calc,
   engine edge cases (NaN sells, unfilled buys), virtual fund determinism/risk gates,
   PM tunables evolution, i18n completeness.

## 7. Known Limitations / Approximations

- Delisted-bond history is Wind-only (Tushare gaps); active+delisted merge assumes
  base equivalence (verified once, 2026-10).
- Live-mode 剩余规模/强赎标记 come from the current jisilu snapshot (static approximation
  for historical replay).
- Execution simulation: T+0 fills at open/close ±slippage; no order-book depth;
  treat replay results as **relative** PM comparison.
- Wind-dependent features (发行进度, 监控快照) degrade gracefully without a Wind terminal.
- 换手率 in cb_prices uses current remain size for all history (approximation).
- cb_holders covers only ~139 disclosed bonds; quarterly.

*Generated 2026-10-19 from full source review. Sizes: 40+ modules, ~15k LOC, 67 tests.*
