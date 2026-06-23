# backtest-trading-framework

## Overview
This repository contains a modular, nine-step quantitative backtesting pipeline designed to rigorously evaluate a systematic moving average crossover strategy. The framework is structured sequentially, ensuring a clean separation of concerns from raw data ingestion through signal generation, execution simulation, and institutional-grade risk analytics. 

By persisting intermediate state via `.parquet` files at each stage, the architecture prevents data leakage, enables rapid iteration on specific modules (like risk sizing or parameter optimization), and maintains strict contextual alignment across all analyzed equities.

## Pipeline Architecture

### Notebook 1: Data Loading & Preprocessing
* **Status:** Complete.
* **Work:** Cleans and standardizes the primary price panel.
* **Output:** `data/processed/*.parquet` (serves as the root input for the entire pipeline).

### Notebook 1b: OHLCV Acquisition
* **Objective:** Closes the data gap required for intra-trade metrics.
* **Work:** Pulls daily Open/High/Low/Close/Volume per ticker over the exact date range as Notebook 1. Runs identical data quality checks, including null handling, duplicate removal, and pre-IPO boundary filtering.
* **Output:** `data/processed/ohlcv_{ticker}.parquet`.
* **Watch-out:** Must align exactly to Notebook 1's trading calendar to prevent silent misalignment during MAE/MFE lookups later in the pipeline.

### Notebook 2: Signal Construction (Section A)
* **Input:** Notebook 1 cleaned Close panel.
* **Work:** Computes parameterized moving averages (SMA default, EMA/WMA selectable via config). Explicitly applies signal lag correction `signal.shift(1)` to prevent look-ahead bias. Calculates a 200-day trend filter flag as an optional add-on column rather than a hard gate, allowing A/B testing in later notebooks.
* **Output:** `data/processed/signals_{ticker}.parquet` (Contains Close, MA_short, MA_long, signal, lagged position, and trend_filter flag).
* **Note:** Moving average window sensitivity analysis is deferred to Notebook 8.

### Notebook 3: Backtest Engine & Trade Sheet (Section B & E1)
* **Input:** Notebook 2 signals and Notebook 1b OHLCV (with Close-only fallback labeled).
* **Work:** Executes the core trade-loop engine. Switchable between fixed percentage and ATR-based stop-losses via a config flag. Applies a 5bps transaction cost and a documented conservative fixed slippage assumption. Computes critical per-trade metrics: exit type, holding days, return percentage, R-multiple, Maximum Adverse Excursion (MAE), Maximum Favorable Excursion (MFE), and entry/exit efficiency.
* **Output:** `data/processed/trades_{ticker}_fixed.parquet` and `trades_{ticker}_atr.parquet`.
* **Watch-out:** Stop-loss and take-profit checks must execute against the daily High/Low, not the Close, to accurately reflect triggered stops.

### Notebook 4: Trade-Level Analytics (Section C)
* **Input:** Fixed-stop trade sheets from Notebook 3.
* **Work:** Calculates expectancy (absolute and per-day), payoff ratio, return distribution skewness/kurtosis, percentile tails, and consecutive win/loss streaks. Computes the Kelly and Half-Kelly fraction (flagging aggressively high outputs). Maps exit-type attribution and generates MAE vs. PnL scatter plots.
* **Output:** `trade_analytics_{ticker}.parquet` and a persisted Kelly fraction per ticker for downstream use.

### Notebook 5: Equity Curves & Portfolio Construction (Section F)
* **Input:** Notebook 2 and 3 outputs.
* **Work:** Generates per-stock daily strategy returns. Implements and documents explicit rebalancing mechanics (daily-rebalanced equal weight). Plots the portfolio equity curve against individual stock curves, a 6-month rolling correlation heatmap, and monthly/yearly contribution stacked bars. Analyzes drawdown synchronization to evaluate true diversification.
* **Output:** `strategy_returns_all.parquet` (wide panel, 4 columns) and `portfolio_returns.parquet`.

### Notebook 6: Performance Metrics (Section D)
* **Input:** `strategy_returns_all.parquet` and buy-and-hold benchmark series.
* **Work:** Calculates Sharpe, Sortino, Calmar, and Omega ratios. Computes parametric and historical VaR/CVaR, 60-day rolling Sharpe, and an extended drawdown analysis (duration, average, threshold counts, and underwater plots). Compares strategy active return and Information Ratio against the buy-and-hold benchmark. Generates monthly return heatmaps and annual bar charts.
* **Output:** `performance_summary.parquet`.
* **Watch-out:** Metrics are reported for each stock and the portfolio side-by-side to highlight how diversification impacts portfolio-level Value at Risk relative to individual assets.

### Notebook 7: Risk Management - Sizing & Stop Calibration (Sections E1-E4, E9-E10)
* **Input:** Notebook 3 ATR/Fixed trade sheets, Notebook 4 Kelly fractions, Notebook 5 return panel.
* **Work:** Conducts side-by-side metric comparisons of ATR-stops versus fixed percentage stops. Recomputes portfolio equity curves utilizing volatility-targeted position sizing and fractional-Kelly sizing. Calculates the full-sample correlation matrix, portfolio volatility, diversification ratio, percentage of time in the market, and time-adjusted CAGR.
* **Output:** `risk_calibration_summary.parquet`.

### Notebook 8: Parameter Optimization & Robustness (Sections E5-E6)
* **Input:** The complete Notebook 2 to Notebook 3 logic pipeline.
* **Work:** Executes a parameter grid search for Stop-Loss/Take-Profit combinations (outputting Sharpe and Calmar heatmaps) and MA-window sensitivities. Attaches explicit overfitting disclaimers to all optimization outputs and recommends out-of-sample or walk-forward validation rather than blind selection of peak values.
* **Output:** `optimization_results.parquet` and heatmap figures.

### Notebook 9: Monte Carlo & Stress Testing (Sections E7-E8)
* **Input:** Notebook 5 and 6 return series.
* **Work:** Runs a 1,000-path resampled bootstrap fan chart with a fixed random seed for reproducibility. Calculates 5th/95th percentile final returns, maximum drawdowns, and the probability of positive annualized returns. Isolates strategy performance during specific historical shocks: the COVID Crash (2020), Rate Hike Selloff (2022), US Debt Ceiling (2011), and China Devaluation (2015).
* **Output:** `monte_carlo_results.parquet` and `stress_test_summary.parquet`.
