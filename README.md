# Moving Average Crossover Backtester

## Project Overview
This repository contains a capstone project focused on algorithmic backtesting. The objective is to design, implement, and evaluate a long-only moving average crossover strategy applied to four major tech equities: Apple (AAPL), Meta (META), Microsoft (MSFT), and Tesla (TSLA) spanning the period 2010 to 2022. 

Additionally, the project constructs an equal-weighted portfolio from these assets and evaluates its risk-adjusted performance using standard quantitative metrics.

## Methodology

### 1. Signal Generation
The strategy utilizes a dual moving average crossover. The parameters are defined as:
* Fast Moving Average: 9-day rolling mean
* Slow Moving Average: 21-day rolling mean

A long signal is generated when the 9-day moving average strictly exceeds the 21-day moving average. The strategy is constrained to long-only positions.

### 2. Portfolio Construction
An equal-weighted portfolio is constructed by grouping the strategy returns of the four individual equities and applying a daily mean across the assets.

### 3. Performance Metrics
The portfolio's performance is evaluated using the following core metrics:

* **Compound Annual Growth Rate (CAGR):** Measures the smoothed annualized return.
$$CAGR = \left( \frac{V_f}{V_i} \right)^{\frac{1}{t}} - 1$$

* **Sharpe Ratio:** Measures the risk-adjusted return relative to a risk-free rate.
$$S = \frac{R_p - R_f}{\sigma_p}$$

* **Maximum Drawdown (MDD):** Measures the largest peak-to-trough drop in the portfolio's equity curve.
$$MDD = \frac{Trough - Peak}{Peak}$$

## Repository Structure
* `data/`: Directory for raw CSV price data (ignored by git).
* `notebooks/`: Contains the primary Jupyter Notebook with the backtesting logic.
* `README.md`: Project documentation.

## Setup and Installation
1. Clone the repository.
2. Place the sample data CSV file into the `data/` directory.
3. Install the required dependencies: `pip install pandas numpy matplotlib`
4. Run the Jupyter Notebook to view the backtest results.
