# EUR/USD 5-Minute Meta-Labeling Walk-Forward Research Notebook


This project develops a statistically defensible event-driven machine learning pipeline for EUR/USD 5-minute data (2019–2022) using alpha-event generation, meta-labeling, rolling walk-forward validation, embargoing, PCA dimensionality reduction, XGBoost classification, top-k probability-based trade selection, and fully out-of-sample backtesting. The workflow begins with trend/pullback-based alpha signals (EMA50/EMA200 trend filter, RSI pullback, volatility filtering), followed by engineered momentum, volatility, range, and session features. Candidate events are meta-labeled based on future forward returns and evaluated using rolling retraining with strict train → embargo → test separation to eliminate look-ahead bias. PCA is fitted only on training folds before XGBoost models are retrained sequentially across the full dataset, producing chronological OOS probability forecasts used for sparse long-only trade selection. The resulting strategy is evaluated through confusion matrices, ROC-AUC analysis, equity curves, Backtesting.py execution simulation, QuantStats tear sheets, and probabilistic/deflated Sharpe ratio diagnostics. Results suggest the presence of weak but potentially genuine predictive structure, while DSR analysis indicates that the observed edge is not yet statistically exceptional after accounting for multiple testing and research overfitting.

## Environment

```text
Python: 3.13.3
numpy: 2.2.6
pandas: 2.3.3
sklearn: 1.8.0
matplotlib: 3.10.9
seaborn: 0.13.2
yfinance: 0.2.65
quantstats: 0.0.81
backtesting: 0.6.5
```

---

## Objective

Develop a statistically defensible event-driven ML trading pipeline for:

- EUR/USD
- 5-minute candles
- September 2019 to September 2022

using:

- alpha-event generation
- meta-labeling
- rolling walk-forward validation
- embargoing
- PCA dimensionality reduction
- XGBoost classification
- top-k signal selection
- out-of-sample backtesting
- probabilistic / deflated Sharpe analysis

---

## Core Research Pipeline

```text
Raw FX data
    ↓
Feature engineering
    ↓
Alpha-event generation
    ↓
Meta-label creation
    ↓
Walk-forward splits
    ↓
Embargoing
    ↓
Scaling
    ↓
PCA
    ↓
XGBoost
    ↓
Top-k trade selection
    ↓
OOS predictions
    ↓
Strategy returns
    ↓
Backtesting.py
    ↓
QuantStats
    ↓
PSR / DSR analysis
```

---

## Data Loading

```python
df = pd.read_csv(CSV_PATH)
```

Using:

```python
N_ROWS = None
```

ensures the full dataset is loaded instead of truncating the data around early 2020.

---

## Alpha Event Generation

Several alpha ideas were explored. The final candidate alpha was:

```python
alpha_signal = (
    df["uptrend"]
    & df["pullback"]
    & df["vol_ok"]
).astype(int)
```

Where:

| Feature | Meaning |
|---|---|
| `uptrend` | EMA50 > EMA200 |
| `pullback` | RSI(14) < 40 |
| `vol_ok` | ATR percentile < 80% |

This creates long-only pullback candidate events.

---

## Feature Engineering

### Base Features

| Feature | Description |
|---|---|
| `ret_1` | 1-bar return |
| `rsi_14` | RSI |
| `atr_pct` | ATR normalized by price |
| `atr_rank` | rolling ATR percentile |
| `ema_50` | fast trend |
| `ema_200` | slow trend |

### Extended Features

#### Multi-Horizon Momentum

```python
ret_3
ret_6
ret_12
ret_24
```

#### Distance from Extremes

```python
dist_high_50
dist_low_50
```

These measure location within the recent rolling range.

#### Volatility Acceleration

```python
atr_change
```

#### Session Features

```python
london_session
ny_overlap
```

These capture FX session structure.

---

## Meta-Labeling

Meta-label target:

```python
future_ret > 0.0005
```

with:

```python
META_BARS = 24
```

Meaning:

```text
Will this alpha event generate >5bp return within the next 24 bars?
```

Since the data is 5-minute candles, 24 bars is roughly 2 hours.

---

## Embargoing

Embargo:

```python
EMBARGO = META_BARS
```

This prevents overlapping label leakage between training events and future test events.

This is critical in financial ML because adjacent event labels often share future information.

---

## Walk-Forward Validation

Used:

```python
TRAIN_SIZE = 800
TEST_SIZE = 200
```

Structure:

```text
Train → Embargo → Test
```

Sequential retraining:

```text
[train][embargo][test]
       shift →
```

This creates out-of-sample predictions without look-ahead bias.

---

## PCA

Pipeline:

```python
StandardScaler
→ PCA(95% variance)
```

PCA is fitted only on the training fold, then applied to the test fold.

This avoids leakage.

---

## PCA Interpretation

Observed structure:

- trend regime
- volatility regime
- momentum clustering
- mean-reversion structure

Typical interpretation:

| PC | Interpretation |
|---|---|
| PC1 | trend / volatility |
| PC2 | volatility regime |
| PC3 | momentum |
| PC4 | mean-reversion |
| PC5+ | residual structure |

The PCA analysis showed that many features are correlated and compress into a few latent market factors.

---

## XGBoost Meta-Model

Configuration:

```python
xgb.XGBClassifier(
    n_estimators=300,
    max_depth=3,
    learning_rate=0.03,
    subsample=0.8,
    colsample_bytree=0.8,
    eval_metric="logloss",
    random_state=42,
    n_jobs=-1,
)
```

The model is retrained fresh on each walk-forward fold.

---

## Signal Construction

Instead of fixed thresholding:

```python
signal = (proba > 0.55)
```

final signal construction used top-k selection.

### Top-k Selection

```python
TOP_K_FRAC = 0.10
```

Only the top 10% highest-confidence candidate events per fold are traded.

This improved:

- precision
- trade quality
- stability

The result suggests that the model is better at ranking opportunities than producing calibrated absolute probabilities.

---

## Out-of-Sample Prediction Aggregation

All fold predictions are concatenated chronologically:

```python
ml_results = pd.concat(results)
```

This produces a true out-of-sample prediction stream across the data range.

---

## Confusion Matrix

Evaluated with:

```python
confusion_matrix(
    ml_results["y_true"],
    ml_results["signal"]
)
```

The meta-labeling + top-k approach produced sparse but higher-quality signals.

Typical behavior:

- fewer false positives
- lower recall
- higher precision

This is common in event-driven trading systems.

---

## ROC Curve

ROC-AUC analysis showed meaningful probability separation between successful and unsuccessful events.

This suggests the model is learning weak but non-random structure.

---

## Strategy Return Construction

Returns:

```python
strategy_ret = future_ret * signal
```

Net returns:

```python
net_ret = strategy_ret - spread_cost * signal
```

with:

```python
SPREAD_COST = 0.00005
```

---

## Equity Curve

Equity curve:

```python
(1 + net_ret).cumprod()
```

Observed behavior:

- upward long-term drift
- sparse event-driven exposure
- fold variability

This is realistic for intraday FX ML systems.

---

## Backtesting.py Integration

Signals are mapped back onto the full 5-minute price series.

Strategy logic:

```text
enter long when ML signal = 1
hold for 24 bars
exit
```

This converts vectorized research returns into an execution-aware backtest.

---

## QuantStats

QuantStats was used for:

- drawdown analysis
- rolling Sharpe
- monthly heatmaps
- distribution diagnostics
- tearsheet generation

Example:

```python
qs.reports.full(
    returns,
    output="meta_label_strategy_report.html",
    title="EURUSD Meta-Label Strategy"
)
```

---

## Sharpe Analysis

Raw annualized Sharpe was very high:

```text
Annualized Sharpe ≈ 13.45
```

This is likely inflated by:

- sparse event structure
- overlapping observations
- serial dependence
- aggressive annualization assumptions

Therefore, annualized Sharpe should not be trusted blindly.

---

## Probabilistic Sharpe Ratio

Corrected PSR implementation uses non-annualized Sharpe inside the PSR formula.

Observed:

```text
PSR ≈ 0.999999
```

Interpretation:

```text
The observed edge is likely not purely random.
```

---

## Deflated Sharpe Ratio

Observed:

```text
DSR ≈ 0
```

Interpretation:

```text
After accounting for multiple testing / research overfitting,
the strategy is not yet statistically exceptional.
```

This is a healthy and realistic diagnostic result.

---

## Key Research Conclusions

### Positive Signs

- genuine probability separation
- out-of-sample walk-forward edge
- improved precision after meta-labeling
- non-random equity drift
- realistic sparse trade structure

### Remaining Weaknesses

- weak absolute edge
- likely feature instability
- simplistic labels
- overlapping-event assumptions
- no triple-barrier labeling yet
- no purged k-fold CV yet

---

## Most Important Insight

The current bottleneck is likely:

```text
better alpha generation
```

not:

```text
more complex ML models
```

The framework itself is already fairly serious:

- walk-forward validation
- embargoing
- PCA
- out-of-sample testing
- top-k ranking
- meta-labeling
- execution backtesting
- PSR / DSR evaluation

---

## Next Research Directions

### 1. Triple-Barrier Labeling

Replace fixed-horizon labels with:

- take-profit barrier
- stop-loss barrier
- time barrier

This is probably the most important next upgrade.

### 2. Better Alpha Events

Potential directions:

- volatility compression breakouts
- session asymmetry
- trend persistence
- local volatility geometry
- distance from local extremes
- higher-timeframe confirmation

### 3. Purged Cross-Validation

Add purged k-fold CV for more robust model selection.

Walk-forward should remain the final backtest method.

### 4. Regime Models

Train or evaluate separate models for:

- London session
- NY overlap
- high-volatility regimes
- low-volatility regimes
- trending regimes
- mean-reverting regimes

FX is highly regime-dependent.

---

## Final Assessment

This project evolved from a simple indicator-ML experiment into a reasonably serious event-driven quantitative research pipeline.

The current system contains:

- proper out-of-sample methodology
- rolling retraining
- leakage mitigation
- realistic statistical diagnostics
- institutional-style research structure

The next major improvement should come from better alpha/event definitions and better labels, not simply from trying more complex classifiers.
