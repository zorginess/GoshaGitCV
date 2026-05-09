# SPY Trend Following Strategy with Volatility Targeting and Walk-Forward Backtesting

## Overview

This project investigates a medium-term time-series momentum strategy on the SPY ETF using moving-average crossover signals, volatility-targeted position sizing, realistic transaction costs, and walk-forward parameter optimization.

The core goal is not necessarily to outperform buy-and-hold SPY in absolute returns, but rather to evaluate whether systematic trend-following techniques can:

- reduce severe equity drawdowns,
- stabilize portfolio volatility,
- maintain reasonable long-term performance,
- and remain robust under realistic implementation constraints.

The project intentionally incorporates:

- lagged signals to avoid look-ahead bias,
- transaction costs,
- participation-based slippage,
- out-of-sample testing,
- and walk-forward optimization.

---

# Strategy Logic

## 1. Moving Average Crossover Signal

The strategy uses a trend-following signal based on two moving averages:

- Short-term moving average
- Long-term moving average

Signal generation:

- Long when short MA > long MA
- Flat or short when short MA < long MA

Both SMA and EMA variants can be tested.

Example parameter combinations:

- 20 / 126
- 50 / 252
- 63 / 300
- 100 / 300

The best-performing regions were generally associated with slower long-term trend filters.

---

# Volatility Targeting

A key component of the project is volatility-targeted position sizing.

Instead of maintaining constant exposure, the strategy dynamically adjusts leverage based on realized volatility:

- Higher volatility → reduced exposure
- Lower volatility → increased exposure

Target annualized volatility:

- 15%

Maximum leverage cap:

- 2x

This significantly stabilizes the equity curve and reduces tail risk.

---

# Transaction Costs and Slippage

The backtest includes multiple layers of implementation realism.

## Brokerage Costs

Fixed brokerage fee:

- 2 basis points per unit turnover

## Bid-Ask Spread

Spread cost:

- 0.32 basis points

## Participation-Based Slippage

The strategy models market impact using participation rate relative to SPY daily traded value.

Assumptions:

- Maximum daily traded capital: $1,000,000
- Maximum market participation: 1% ADV
- 1% ADV participation corresponds to approximately 10 bps slippage

This prevents unrealistic execution assumptions and forces the strategy to remain scalable.

---

# Walk-Forward Backtesting

A walk-forward optimization framework is used to reduce overfitting.

Process:

1. Train on historical window
2. Select best MA parameters
3. Trade next unseen period
4. Roll forward and repeat

Typical configuration:

- 5 years training
- 1 year testing

Parameter selection is based on a score balancing:

- Sharpe ratio
- drawdown penalty

This simulates a more realistic research and deployment pipeline.

---

# Key Findings

## 1. Buy-and-Hold Still Produces Higher Returns

The strategy generally underperforms passive SPY in total return and CAGR.

This is consistent with modern equity market structure:

- persistent long-term upward drift,
- rapid post-crisis recoveries,
- and strong passive equity beta.

Trend-following systems often sacrifice upside participation in exchange for risk reduction.

---

## 2. Significant Drawdown Reduction

The most important result of the project is substantial drawdown reduction.

### Approximate Results

| Strategy | Max Drawdown |
|---|---|
| Buy-and-Hold SPY | Worse than -50% |
| Walk-Forward Trend Strategy | Approximately -30% |

The strategy materially reduces portfolio crashes during:

- Dot-com collapse
- 2008 Global Financial Crisis
- COVID regime shocks

while still maintaining meaningful long-term growth.

This demonstrates that trend-following combined with volatility targeting can behave as a risk-management overlay rather than a pure return-maximization engine.

---

## 3. Robustness Across Parameters

Performance was not concentrated in a single parameter pair.

Multiple neighboring regions produced similar results:

- 50 / 252
- 63 / 300
- 100 / 300

This is encouraging because it suggests the strategy captures a broader market behavior rather than fitting noise.

---

## 4. Long-Only Performs Better Than Long-Short

The project found that long-only trend-following generally performs better than long-short trend-following on SPY. (Even without the price/availability of shortening a position)

Reason:

- Equity indices possess strong structural upward drift.
- Shorting broad equity indices over long horizons is difficult.

Therefore:

- Long/flat trend systems are often more robust than long/short systems for equity indices.

---

# Drawdown Interpretation

One of the central conclusions is:

> Trend-following on SPY may not maximize returns, but it can substantially improve risk characteristics.

Reducing maximum drawdown from greater than 50% toward approximately 30% is extremely meaningful from a portfolio construction perspective.

Large drawdowns:

- increase behavioral risk,
- reduce compounding efficiency,
- and create severe recovery requirements.

For example:

- a 50% loss requires a 100% recovery,
- while a 30% loss requires only ~43% recovery.

Thus, lower drawdowns can improve long-term capital survivability even when CAGR is somewhat lower.

---


