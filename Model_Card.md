# Model Card — Moving Average Crossover Backtesting System

> **Developer:** Backtesting Trading Strategies Project  
> **Model Date:** 2025  
> **Model Version:** 1.0 (Initial Backtest)  
> **Universe:** AAPL, META, MSFT, TSLA (US Equities)  
> **Data Range:** 2010-01-01 – 2025-12-31 (NYSE calendar-aligned)

---

## Model Description

This model card documents an end-to-end algorithmic trading backtesting pipeline. The system spans nine sequential modules — from raw data ingestion through to Monte Carlo simulation — each constituting a distinct model or engine. All modules together form a single reproducible research artefact whose purpose is to evaluate a momentum-based moving average crossover strategy on a four-asset US equity portfolio.

---

### Module Architecture Overview

| # | Notebook | Module | Role |
|---|----------|--------|------|
| 1 | `01_Data_Loading___Preprocessing` | **Data Pipeline** | Ingest, clean, mask, and serialise OHLCV data |
| 2 | `02_Signal_Construction` | **Signal Generation Model** | Compute MA crossover signals (SMA & EMA) with trend filter |
| 3 | `03_Backtest_Engine___Trade_Sheet` | **Backtest & Execution Engine** | Simulate trade entries, exits, stops, and costs; produce trade sheets |
| 4 | `04_Trade_Level_Analytics` | **Trade Analytics Engine** | Compute per-trade statistics, Kelly criterion, distribution metrics |
| 5 | `05_Equity_Curves___Portfolio_Construction` | **Portfolio Construction Model** | Aggregate assets into an equal-weight portfolio; generate equity curves |
| 6 | `06_Performance_Metrics` | **Performance Metrics Engine** | Calculate risk-adjusted performance metrics against a Buy & Hold benchmark |
| 7 | `07_Risk_Management_Sizing___Stop_Calibration` | **Risk Management Engine** | Volatility targeting overlay and fractional Kelly sizing |
| 8 | `08_Parameter_Optimization___Robustness` | **Optimisation & Robustness Engine** | Grid-search MA and SL/TP parameters; walk-forward out-of-sample validation |
| 9 | `09_Monte_Carlo___Stress_Testing` | **Monte Carlo & Stress Testing Engine** | Bootstrap simulation (10,000 paths) and historical stress scenario analysis |

---

### Module 1 — Data Pipeline

**Input:** Yahoo Finance OHLCV data for AAPL, META, MSFT, TSLA.

**Output:** Parquet files per ticker (`ohlcv_{ticker}.parquet`) with an `is_valid_day` validity mask column and a machine-readable `metadata.json` audit trail.

**Architecture:** Calendar alignment to the NYSE schedule (via `pandas_market_calendars`) to eliminate phantom non-trading days. A tiered masking rule marks rows as invalid if fewer than three OHLCV columns are present ("fatal" days), while rows with partial data (three or four valid columns) are forward-filled and preserved. META data prior to its IPO date (18 May 2012) is explicitly null-masked. Dividend and split adjustments are applied via `yfinance auto_adjust=True`.

---

### Module 2 — Signal Generation Model

**Input:** Cleaned OHLCV parquet files.

**Output:** Signal parquet files (`signals_{ticker}.parquet`) containing SMA/EMA crossover signals, trend filter flags, final lagged positions, and daily strategy returns for both MA variants.

**Architecture:** A dual-path momentum model implementing both Simple Moving Average (SMA) and Exponential Moving Average (EMA) crossovers under identical parameterisation. A 200-period long-term trend filter acts as an independent gate: positions are only opened when the closing price sits above the trend average. A mandatory one-period lag (`shift(1)`) is applied to all final position columns so that signals generated at day *t* are only actionable at day *t+1*, eliminating look-ahead bias by design.

| Parameter | Value |
|-----------|-------|
| Short MA window | 9 periods |
| Long MA window | 21 periods |
| Trend filter window | 200 periods |
| Signal lag | 1 trading day |
| Variants | SMA and EMA (both evaluated) |

**Signal Logic:**

```
raw_signal      = 1 if short_MA > long_MA else 0
trend_gate      = 1 if Close > MA_200 else 0
final_position  = shift(raw_signal × trend_gate, 1)
```

---

### Module 3 — Backtest & Execution Engine

**Input:** Signal parquet files.

**Output:** Trade-sheet parquet files (`trades_{ticker}_{mode}_{strat}.parquet`) with one row per completed round-trip trade.

**Architecture:** An event-driven engine that iterates row-by-row through the signal series. Entries are executed at the next day's open price following a signal. Two stop-loss regimes are implemented and evaluated independently:

- **Fixed Stop:** Entry price × (1 − 2%) — a constant percentage stop independent of volatility.
- **ATR Stop:** Entry price − (14-period ATR × 2.0 multiplier) — a volatility-adaptive stop that widens in high-volatility environments and tightens in low-volatility periods.

Exits are triggered by whichever condition fires first: a stop-loss breach (evaluated against the intraday low) or a signal reversal to zero (evaluated at the open). Transaction costs (5 bps per leg) and slippage (1 bp per leg) are deducted from each round-trip.

Per-trade fields captured:

| Field | Description |
|-------|-------------|
| Return % | Net return after execution drag |
| DS-Risk Multiple | Return divided by downside risk (entry to stop distance) |
| MAE | Maximum Adverse Excursion — lowest intraday penetration while open |
| MFE | Maximum Favourable Excursion — highest intraday gain while open |
| Entry Efficiency | Closeness of entry price to the trade's low (0 = ideal) |
| Exit Efficiency | Closeness of exit price to the trade's high (0 = ideal) |
| Exit Type | Stop-loss triggered (`SL`) or signal reversal (`Signal`) |

**Execution matrix:** 4 tickers × 2 stop modes × 2 MA variants = **16 trade sheets** generated.

---

### Module 4 — Trade Analytics Engine

**Input:** Trade sheets (baseline: ATR stop, EMA variant).

**Output:** `trade_analytics_summary.parquet` and `kelly_fractions.json`.

**Architecture:** A statistical summary engine that aggregates trade-level data into distributional and sizing metrics. The Kelly criterion is computed per-ticker and capped with a Half-Kelly divisor for practical safety. Key outputs:

| Metric | Description |
|--------|-------------|
| Win Rate | Fraction of trades with positive net return |
| Profit Factor | Total gross wins ÷ total gross losses |
| Payoff Ratio | Average win ÷ average loss magnitude |
| Expectancy | Expected return per trade (probability-weighted) |
| Expectancy / Day | Expectancy normalised by average holding period |
| Kelly Fraction | Optimal bet size per the Kelly criterion |
| Half Kelly | Conservative Kelly ÷ 2 allocation recommendation |
| Skewness | Asymmetry of the return distribution |
| Kurtosis | Fat-tail measure of the return distribution |
| Max Win / Loss Streak | Longest consecutive profitable or losing run |
| 5th / 95th Percentile | Tail return bounds |

A sizing warning is raised if the full Kelly fraction exceeds 25%, signalling extreme volatility risk.

---

### Module 5 — Portfolio Construction Model

**Input:** Signal parquet files.

**Output:** `strategy_returns_all.parquet`, `portfolio_returns.parquet`.

**Architecture:** A daily-rebalanced equal-weight portfolio (25% per asset). Assets not in an active long trade are treated as holding cash at 0% return, enforcing strict capital isolation. Execution drag is applied on days where the position changes (entry or exit). Multi-timeframe equity curves are generated (all-time, 5Y, 1Y, 1M), and a 126-trading-day (≈ 6-month) rolling pairwise correlation heatmap is produced to monitor diversification stability over time. An annual contribution breakdown decomposes portfolio return into per-asset contributions (each asset's return ÷ 4).

---

### Module 6 — Performance Metrics Engine

**Input:** Portfolio and strategy returns; Buy-and-Hold benchmark constructed from raw price series.

**Output:** `performance_summary.parquet` — a metrics table covering all four individual assets plus the equal-weight portfolio.

**Architecture:** A library of annualised risk-adjusted performance functions evaluated identically across strategy and benchmark:

| Metric | Formula / Method |
|--------|-----------------|
| Sharpe Ratio | (Mean excess return / Std dev) × √252 |
| Sortino Ratio | (Mean excess return / Downside std) × √252 |
| Calmar Ratio | Annualised return / Absolute Max Drawdown |
| Omega Ratio | Probability-weighted gains / Probability-weighted losses above threshold |
| Historical VaR (95%) | 5th percentile of the empirical return distribution |
| Parametric VaR (95%) | Normal distribution approximation using mean and std |
| CVaR (95%) | Mean return conditional on being below the 5th percentile |
| Tail Ratio | 95th percentile return / Absolute 5th percentile return |
| Max Drawdown | Peak-to-trough decline from the cumulative return series |
| Avg DD Duration | Mean number of consecutive days spent below a drawdown threshold |
| Days DD > 5/10/15% | Count of trading days with drawdown exceeding each threshold |
| Active Return | Annualised strategy return minus annualised benchmark return |
| Information Ratio | Active return / Tracking error (annualised) |

---

### Module 7 — Risk Management Engine

**Input:** Portfolio returns; Kelly fractions from Module 4.

**Output:** `risk_calibration_summary.parquet`.

**Architecture:** Three complementary sizing overlays evaluated side by side:

1. **Static Equal-Weight Baseline:** 25% per asset, no dynamic adjustment.
2. **Volatility Targeting Overlay:** Target annualised portfolio volatility of 15%. A 20-day rolling standard deviation is annualised and used to scale daily position size. Leverage is bounded between 0.1× and 2.0× for safety. The scalar is lagged by one day to avoid look-ahead.
3. **Fractional Kelly (Half Kelly) Sizing:** Per-asset allocation is scaled by the Half-Kelly fraction derived in Module 4.

Additionally, the full-sample correlation and covariance matrix is computed across strategy returns, and a **Diversification Ratio** is reported (weighted average individual volatility ÷ realised portfolio volatility). A ratio greater than 1.0 confirms that portfolio volatility benefits from imperfect correlation between assets.

---

### Module 8 — Optimisation & Robustness Engine

**Input:** Raw OHLCV and signal data for AAPL (optimisation universe).

**Output:** `optimization_ma_results.parquet`, `optimization_sltp_results.parquet`.

**Architecture:** A two-stage grid search implemented on a 70% in-sample / 30% out-of-sample chronological split.

**Stage 1 — MA Window Grid Search:**

| Short MA candidates | Long MA candidates |
|--------------------|--------------------|
| 5, 9, 12, 15, 20 | 21, 50, 100, 150, 200 |

Valid pairs only (short < long): 20 valid combinations evaluated.

**Stage 2 — Stop-Loss / Take-Profit Grid Search** (fixed at 9/21 MA):

| Stop Loss levels | Take Profit levels |
|-----------------|-------------------|
| 2%, 4%, 6%, 8%, 10% | 5%, 10%, 15%, 20%, 25%, 30% |

Objective function: Sharpe Ratio and Calmar Ratio (both reported; selection is left to the researcher to avoid single-metric overfitting).

**Walk-Forward Validation:** The in-sample optimal parameters are applied to the held-out out-of-sample period. Sharpe Ratio degradation is computed:

```
Degradation = (IS Sharpe − OOS Sharpe) / IS Sharpe
```

Degradation > 40% triggers a severe overfitting warning. Degradation between 0% and 40% is classified as normal regime decay. Negative degradation (OOS > IS) indicates out-of-sample outperformance.

---

### Module 9 — Monte Carlo & Stress Testing Engine

**Input:** Historical daily portfolio returns from Module 5.

**Output:** `monte_carlo_results.parquet`, `stress_test_summary.parquet`.

**Architecture:** Two complementary tail-risk frameworks:

**Bootstrap Monte Carlo Simulation:** 10,000 independent paths of 252 trading days are generated by resampling historical daily returns with replacement (random seed: 42 for reproducibility). This preserves the empirical return distribution (including skewness and fat tails) while assuming return independence across days. Outputs: 5th / 50th / 95th percentile terminal values and probability of positive return over the horizon.

**Historical Stress Testing:** The portfolio return series is sliced across three documented market stress windows and evaluated independently:

| Event | Window |
|-------|--------|
| COVID-19 Crash | 19 Feb 2020 – 23 Mar 2020 |
| 2022 Fed Rate Hike Regime | 01 Jan 2022 – 31 Dec 2022 |
| 2015 China Devaluation Shock | 11 Aug 2015 – 25 Aug 2015 |

Per-event outputs: Total Return, Max Drawdown, and Annualised Volatility.

---

## Performance

Performance is evaluated at the portfolio level (4-asset equal-weight) against a Buy-and-Hold benchmark of the same assets. The primary measurement datasets are the full 2010–2025 out-of-sample historical returns; no in-sample fitted parameters are used in the main performance report (optimisation is compartmentalised in Module 8).

**Primary Performance Metrics Reported (Module 6):**

| Metric | Strategy | Buy & Hold |
|--------|----------|-----------|
| Annualised Return | Computed from daily portfolio returns | Computed from raw price changes |
| Sharpe Ratio | Risk-free rate = 0 (conservative) | Same |
| Sortino Ratio | Downside deviation only | Same |
| Calmar Ratio | Annualised return / Max DD | Same |
| Max Drawdown | Peak-to-trough from cumulative curve | Same |
| CVaR (95%) | Mean return in worst 5% of days | Same |
| Information Ratio | Active return / Tracking error | — |

**Monte Carlo Summary (Module 9):**

| Percentile | Terminal Value ($1 invested) |
|------------|------------------------------|
| 5th (Worst Case) | Computed from 10,000 bootstrap paths |
| 50th (Median) | Computed from 10,000 bootstrap paths |
| 95th (Best Case) | Computed from 10,000 bootstrap paths |
| Probability of Positive Return | Fraction of 10,000 paths ending above $1 |

**Stress Event Performance (Module 9):**

| Event | Total Return | Max Drawdown | Annualised Volatility |
|-------|-------------|-------------|----------------------|
| COVID-19 (Feb–Mar 2020) | Computed from live returns | Computed | Computed |
| 2022 Rate Hike Regime | Computed from live returns | Computed | Computed |
| 2015 China Devaluation | Computed from live returns | Computed | Computed |

**Robustness Validation (Module 8, AAPL only):**

In-sample and out-of-sample Sharpe ratios are compared on a 70/30 chronological split. Any degradation above 40% is flagged as a sign of overfitting to the historical training window.

> **Note:** Precise numerical values are populated at runtime when notebooks are executed against live data. The table structure above documents the metrics and their sources; final figures depend on the current Yahoo Finance data pull.

---

## Limitations

**Data Coverage:** The universe is restricted to four large-cap US technology and consumer stocks (AAPL, META, MSFT, TSLA). These are among the best-performing equities of the 2010–2025 period, introducing a significant **survivorship bias**: the results cannot be generalised to the broader market, sector-neutral portfolios, or non-US equities.

**Signal Simplicity:** The strategy uses only price-derived features (moving averages). No fundamental data, macro indicators, sentiment signals, or order-flow information is incorporated. The model is intentionally rule-based and interpretable, not predictive in a machine-learning sense.

**Benchmark Construction:** The Buy-and-Hold benchmark uses the same four tickers without any rebalancing. This is not an independent, market-representative benchmark (e.g., S&P 500 or a sector ETF), making relative performance claims context-dependent.

**Transaction Cost Modelling:** Costs are modelled as flat rates (5 bps transaction cost, 1 bp slippage). In practice, costs are volume- and liquidity-dependent, and for TSLA in particular, intraday volatility could produce substantially higher slippage during fast markets.

**Look-Ahead Bias Mitigations Are Necessary but Not Sufficient:** The one-day signal lag removes the primary source of look-ahead bias. However, the ATR indicator uses a 14-period lookback; during the first 14 days of any new data window, ATR values are undefined, and the engine falls back to Fixed stop mode implicitly.

**Single-Asset Optimisation:** Parameter grid search (Module 8) is run exclusively on AAPL. Parameter stability across META, MSFT, and TSLA is assumed but not independently validated in this version.

**Monte Carlo Independence Assumption:** The bootstrap simulation resamples returns independently across days. This ignores autocorrelation, volatility clustering (GARCH-like behaviour), and regime persistence — all known features of equity return distributions. The simulation will underestimate the probability and duration of sustained drawdown periods.

**Fixed Portfolio Weights:** The portfolio is equal-weighted and daily-rebalanced. Real implementation would incur significant rebalancing costs at the daily frequency, which are not fully captured by the 6 bps per-change drag assumption.

**No Short Selling:** The model is long-only. It holds cash (0% return) when no signal is active, meaning it does not monetise downtrends, giving Buy-and-Hold an advantage in strongly trending bull markets.

**META Pre-IPO Gap:** META data before 18 May 2012 is null-masked. Portfolio results for the 2010–2012 period effectively represent a three-asset portfolio, slightly understating diversification costs and historical drawdowns.

---

## Trade-offs

**SMA vs. EMA:**
EMA responds faster to recent price changes than SMA, generating slightly earlier entries and exits. This reduces lag at the cost of higher signal noise and more frequent false crossovers. SMA is smoother and more stable in sideways markets but can be slow to capture trend reversals. The system evaluates both but uses EMA as the baseline for downstream analytics.

**Fixed Stop vs. ATR Stop:**
Fixed percentage stops (2%) provide predictability and ease of capital planning but are blind to current market volatility — too tight in high-volatility periods, too loose in low-volatility periods. ATR stops dynamically adjust but will widen during stress events precisely when tighter risk management is most valuable. ATR stops typically produce fewer stop-outs at the cost of larger individual drawdowns per trade.

**Trend Filter On vs. Off:**
The 200-period trend filter significantly reduces the number of signals generated, filtering out trades in downtrending markets. This improves the win rate and reduces exposure to prolonged bearish regimes but also reduces total trades and can cause the strategy to miss early-stage recovery moves.

**Equal-Weight vs. Volatility-Targeted Allocation:**
Equal-weight is simple, transparent, and requires no additional model assumptions. Volatility targeting introduces a dynamic leverage layer that can improve risk-adjusted returns during calm periods by increasing exposure, but this leverage is pro-cyclical — it reduces exposure during high-volatility drawdowns but may amplify losses if the de-leveraging is delayed relative to a fast-moving crash.

**Kelly Criterion Sizing:**
Full Kelly maximises long-run geometric growth in theory but produces extreme drawdowns in practice. Half-Kelly halves both the expected growth rate and the volatility of the sizing path, which is considered a pragmatic compromise. For tickers with Kelly fractions above 25%, even Half-Kelly may exceed practical risk tolerances.

**In-Sample Optimisation vs. Fixed Parameters:**
Grid-searching MA windows and SL/TP levels maximises in-sample Sharpe but risks curve-fitting to historical market noise. The walk-forward validation in Module 8 attempts to quantify this degradation; however, a single 70/30 split is itself subject to regime dependency. The baseline strategy uses fixed, round-number parameters (9/21 MA) that were pre-specified rather than optimised, which is the more conservative and defensible choice for a first-version backtest.

**Bootstrap Monte Carlo vs. Parametric Simulation:**
Bootstrap resampling preserves the empirical shape of the return distribution (fat tails, skewness) and makes no Gaussian assumptions. Its weakness is that it cannot generate outcomes outside the range of the observed historical sample — extreme tail events not present in 2010–2025 data (e.g., a multi-year depression-level drawdown) will be structurally absent from the simulation.

**Long-Only vs. Long/Short:**
The long-only constraint limits the strategy's ability to extract alpha in bear markets. However, it substantially simplifies execution, reduces margin requirements, eliminates short-squeeze exposure, and is appropriate for an initial backtesting version. A long/short extension would require independent signal design for short entries and separate risk calibration.

---

## Intended Use

**Primary intended use:** Academic and educational evaluation of momentum-based trading strategies on US large-cap equities. Suitable for coursework in algorithmic trading, quantitative finance, and systematic strategy research.

**Primary intended users:** Finance students and early-stage quantitative researchers learning backtesting methodology.

**Out-of-scope uses:** This system is not intended for live trading, real-money deployment, or regulatory reporting. The transaction cost model, universe selection, and single-split validation framework are insufficient for production use. The system should not be used to make investment decisions for third parties.

---

## Ethical Considerations

The strategy is entirely price-based and does not use personal data, protected characteristics, or non-public information. All price data is sourced from public markets via Yahoo Finance. There are no fairness or demographic considerations applicable to this system in its current form.

The primary ethical risk is **overfitting misrepresentation**: presenting backtest results as forward-looking predictions without disclosing the survivorship bias inherent in the ticker selection (AAPL, MSFT, META, TSLA are among the strongest-performing equities of the backtest period). Results should always be accompanied by this context.

---

## Caveats and Recommendations

- Extend the universe to a broader, pre-determined stock universe (e.g., S&P 500 constituents with point-in-time membership data) to eliminate survivorship bias before drawing generalised conclusions.
- Replace the single 70/30 walk-forward split with a **rolling walk-forward** scheme (multiple folds) to produce a distribution of OOS Sharpe ratios rather than a single point estimate.
- Incorporate realistic **market impact modelling** for TSLA, whose intraday liquidity can vary significantly, and which is the most volatile name in the universe.
- Consider an **independent benchmark** (e.g., SPY total return) for relative performance assessment instead of a Buy-and-Hold of the same four assets.
- Extend Monte Carlo to **block bootstrap** or a **GARCH-based simulation** to better capture volatility clustering and regime persistence.
- The META pre-IPO exclusion should be documented consistently in any published results citing the 2010–2025 return history.
- Before extending to live trading, a separate **transaction cost sensitivity analysis** should quantify how performance degrades at higher cost levels (e.g., 15 bps, 25 bps) to establish a minimum edge threshold.
