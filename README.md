# Long–Short Multi-Asset Research and Portfolio Optimization Framework

An end-to-end quantitative research project for constructing, validating, and optimizing systematic long–short portfolios across a universe of 37 exchange-traded funds (ETFs).

The framework connects four stages of the investment process:

1. signal construction and validation;
2. return prediction with multiple model families;
3. residual covariance estimation;
4. constrained mean–variance portfolio optimization.

All model and portfolio parameters are selected using chronological training and validation samples. The test sample is reserved for the final out-of-sample evaluation.

> **Research status:** This repository is an educational research prototype, not a production trading system or investment recommendation.

## Project Overview

The project begins with a reusable multi-asset backtesting engine and gradually expands into a complete modeling and portfolio-construction pipeline.

```text
Market Data
    ↓
Signal Engineering
    ↓
Train / Validation / Test Split
    ↓
Model Comparison
    ↓
Expected-Return Predictions
    ↓
Residual Covariance Estimation
    ↓
Mean–Variance Optimization
    ↓
Validation-Based Risk Selection
    ↓
Final Out-of-Sample Test
```

### Main Research Questions

- Can momentum-based technical signals predict future five-day ETF returns?
- Does risk adjustment or MACD smoothing improve the signals?
- Which predictive model generalizes best according to validation MSE?
- Can residual covariance estimation and portfolio constraints improve the risk profile of the selected model?
- What level of risk aversion provides the preferred validation risk–return trade-off?

## Data and Portfolio Setting

| Item | Setting |
|---|---|
| Asset universe | 37 ETFs |
| Initial capital | $1,000,000 |
| Prediction target | Future five-day return |
| Rebalancing frequency | Every five trading days |
| Execution-price proxy | TWAP = (Open + Close) / 2 |
| Portfolio structure | Long–short, approximately dollar neutral |
| Maximum gross exposure | 2.0 |
| Maximum absolute asset weight | 20% |

The raw market data are not included in this repository. Users must provide appropriately licensed OHLCV data and update the notebook's data-loading path before execution.

## Research Pipeline

### 1. Multi-Asset Backtesting

The initial single-asset logic was generalized into a common backtesting framework for all 37 ETFs. The engine tracks:

- target portfolio weights;
- target shares;
- current holdings;
- cash;
- daily profit and loss;
- net asset value;
- cumulative returns and risk statistics.

Two baseline momentum portfolios were used to test the framework:

- **Portfolio 1:** 150% long exposure to positive-momentum assets and 50% short exposure to negative-momentum assets;
- **Portfolio 2:** long the top two-thirds and short the bottom one-third of assets by cross-sectional momentum rank.

These early full-sample backtests produced cumulative returns of 84.59% and 66.17%, respectively. They are framework diagnostics rather than final out-of-sample results.

### 2. Signal Engineering

The feature set includes four technical signals.

#### Momentum

```math
MOM_{i,t}=\frac{1}{21}\sum_{k=1}^{21}r_{i,t-k}
```

#### Risk-Adjusted Momentum

```math
MOMRA_{i,t}=\frac{MOM_{i,t}}{\sigma_{i,t}}\sqrt{21}
```

#### MACD

```math
MACD_{i,t}=\frac{EMA_{short,i,t}-EMA_{long,i,t}}{Price_{i,t}}
```

#### Smoothed MACD

An additional exponentially weighted moving average is applied to the raw MACD signal.

Signal behavior is evaluated through:

- fixed signal intervals;
- 252-day rolling time-series quantiles;
- daily cross-sectional quintiles;
- long–short portfolio backtests;
- future-return averages and prediction IC.

### 3. Market Regimes and Residualization

VIX- and MOVE-based regimes are introduced to study whether momentum performance changes across market-risk environments.

The project also estimates rolling VIX beta and momentum-factor beta. A daily cross-sectional regression removes the component of momentum beta associated with VIX exposure:

```math
MOM\ Beta_{i,t}=a_t+b_t(VIX\ Beta_{i,t})+Residual_{i,t}
```

The residual produced a positive top-minus-bottom spread of approximately 10.07 basis points. However, its regression coefficient was not statistically significant:

| Statistic | Result |
|---|---:|
| Residual coefficient | 0.002251 |
| p-value | 0.202495 |
| 95% confidence interval | [−0.001211, 0.005713] |
| R² | 0.000395 |

The result is therefore interpreted as weak descriptive evidence rather than a robust standalone factor.

## Look-Ahead-Safe Modeling Framework

The data are divided chronologically into training, validation, and test periods. Random splitting is not used.

Several controls are implemented to reduce forward-looking bias:

- signals are delayed before prediction or portfolio formation;
- future returns are used only as target variables;
- `future_end_date` prevents five-day labels from crossing sample boundaries;
- winsorization thresholds are estimated only on Train;
- means and standard deviations for standardization are estimated only on Train;
- minimum and maximum values for normalization are estimated only on Train;
- Validation selects models and hyperparameters;
- Test is reserved for final evaluation.

### Preprocessing

Each feature receives three transformations:

1. **Winsorization:** clipping at the Train 1st and 99th percentiles;
2. **Standardization:** Train-based z-score transformation;
3. **Normalization:** Train-based min–max scaling.

The standardized features are used by the prediction models.

## Model Comparison

The framework compares three required model families.

| Model family | Implementation | Purpose |
|---|---|---|
| Equal Weights benchmark | Equal-weighted standardized signals with Train-only linear calibration | Simple benchmark |
| Regression model | Lasso | Regularized linear prediction and feature selection |
| Tree model | Random Forest | Nonlinear effects and feature interactions |

In this project, **Equal Weights** means equal weights across the four standardized signals. It does not mean equal weights across the 37 portfolio assets.

### Lasso

Lasso estimates coefficients by minimizing:

```math
\min_{\beta}\left[\frac{1}{2N}\sum_{i=1}^{N}(y_i-X_i\beta)^2+\alpha\sum_j|\beta_j|\right]
```

The best penalty parameter is selected by Validation MSE:

```text
Best alpha = 0.0000221222
```

The fitted coefficients indicate negative short-horizon momentum, positive smoothed-MACD exposure, and little incremental information from MOMRA or raw MACD after controlling for the other features.

### Random Forest

The Random Forest search evaluates:

- `max_depth`: 3, 5, and 8;
- `min_samples_leaf`: 20, 50, and 100;
- `n_estimators`: 200;
- `max_features`: `sqrt`.

The selected specification uses `max_depth=3` and `min_samples_leaf=100`.

### Validation Results

| Model | Validation MSE | Validation MAE | RMSE | Prediction IC |
|---|---:|---:|---:|---:|
| Random Forest | **0.000918108816** | 0.0204177997 | 3.030031% | 0.046442 |
| Lasso | 0.000918706932 | **0.0204032295** | 3.031018% | 0.052733 |
| Equal Signals | 0.000918893801 | 0.0204104337 | 3.031326% | **0.087557** |

Random Forest is selected because it has the lowest Validation MSE, which is the pre-specified model-selection criterion.

The results also demonstrate that the lowest MSE does not necessarily imply the highest IC or the strongest standalone portfolio performance. Point-forecast accuracy and cross-sectional ranking quality are related but distinct objectives.

### Selected Model Test Prediction

| Metric | Random Forest Test Result |
|---|---:|
| MSE | 0.000978356450 |
| MAE | 0.018856095254 |
| RMSE | 3.127869% |
| Prediction IC | −0.009378 |

The slightly negative Test IC indicates weak out-of-sample ranking generalization. The model is not replaced after observing this result because doing so would use the Test sample for model selection.

## Residual Covariance Estimation

Training residuals from the selected Random Forest are calculated as:

```math
e_{i,t}=y_{i,t}-\hat{y}_{i,t}
```

The residuals are pivoted into a date-by-asset matrix. Their cross-sectional covariance produces a 37 × 37 risk matrix:

```math
\hat{\Sigma}=Cov(e_t)
```

This covariance matrix represents the joint risk that remains after accounting for the model's expected-return predictions.

## Mean–Variance Portfolio Optimization

For each rebalance date, the selected model generates a 37-dimensional expected-return vector. The optimizer combines this vector with the 37 × 37 residual covariance matrix.

The objective is:

```math
\min_w \left(\frac{1}{2}\lambda w^T\hat{\Sigma}w-w^T\hat{\mu}\right)
```

where:

- `w` is the portfolio-weight vector;
- `μ̂` is the predicted-return vector;
- `Σ̂` is the residual covariance matrix;
- `λ` is the risk-aversion parameter.

### Portfolio Constraints

```math
\sum_i w_i=0
```

```math
\sum_i |w_i|\leq 2
```

```math
|w_i|\leq 0.20
```

Auxiliary variables are introduced to linearize the absolute-value gross-exposure constraint. Equality and inequality constraints are passed separately to the numerical optimizer.

## Risk-Preference Selection

Risk aversion is selected only on Validation. The primary selection metric is Validation Sharpe ratio, with maximum drawdown used as a secondary consideration.

| Risk Aversion | Return | Annualized Volatility | Sharpe | Maximum Drawdown |
|---:|---:|---:|---:|---:|
| 1 | −5.23% | 16.72% | −0.2460 | −18.02% |
| 5 | −2.29% | 11.07% | −0.1587 | −12.45% |
| 10 | 1.06% | 8.17% | 0.1730 | −8.65% |
| 20 | 3.31% | 6.19% | 0.5694 | −5.10% |
| 50 | 2.70% | 4.21% | **0.6681** | **−2.77%** |

The final selected risk-aversion parameter is:

```text
Risk aversion = 50
```

## Final Out-of-Sample Results

After the model, covariance estimator, constraints, and risk preference are frozen, the full pipeline is applied to Test.

| Metric | Final Test Result |
|---|---:|
| Selected prediction model | Random Forest |
| Selected risk aversion | 50 |
| Total return | **3.52%** |
| Annualized volatility | **4.85%** |
| Sharpe ratio | **0.7521** |
| Maximum drawdown | **−4.60%** |
| Average gross exposure | 2.00 |
| Maximum gross exposure | 2.00 |

These results describe one historical out-of-sample period and should not be interpreted as evidence of guaranteed future performance.

## Debugging and Data-Quality Lessons

### Corporate-Action Adjustment

An implausible one-day portfolio increase of approximately 8.41% appeared on December 5, 2025. Daily-return inspection and asset-level PnL attribution showed concentrated contributions from XLB, XLK, XLY, XLE, and XLU.

The move was traced to 2-for-1 ETF share splits. Without adjustment, the backtester interpreted the mechanical price change as a genuine return and did not compensate the share count.

The historical data were corrected by:

- dividing pre-split OHLC prices by two;
- multiplying pre-split volume by two;
- rebuilding all signals, models, residuals, covariance matrices, and backtests.

The false portfolio jump disappeared after the correction.

### Optimization Convergence

The initial SLSQP implementation sometimes reached its iteration limit because the gross-exposure constraint contained a nondifferentiable absolute-value function.

The problem was repaired by introducing auxiliary variables:

```math
u_i\geq w_i,\qquad u_i\geq-w_i,\qquad \sum_i u_i\leq2
```

This converts the gross-exposure requirement into linear inequalities and improves numerical stability.

### Notebook Execution Order

Errors such as `evaluate_model is not defined` and `best_model_name is not defined` occurred when dependent cells were executed out of order. The final notebook uses consistent function names and should be executed from top to bottom after restarting the kernel.

## Repository Structure

The recommended public repository structure is:

```text
long-short-multi-asset-framework/
├── README.md
├── LS_MA_Quant_Project.ipynb
├── requirements.txt
├── data/
│   └── README.md
└── images/
    └── final_test_performance.png
```

Only `README.md` and the notebook are required to review the project. The additional files improve reproducibility and presentation quality.

## Installation

Clone the repository:

```bash
git clone https://github.com/YOUR-USERNAME/long-short-multi-asset-framework.git
cd long-short-multi-asset-framework
```

Create and activate a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate
```

On Windows:

```bash
.venv\Scripts\activate
```

Install the required packages:

```bash
pip install -r requirements.txt
```

Start Jupyter:

```bash
jupyter notebook
```

Open `LS_MA_Quant_Project.ipynb`, update the data-loading path, restart the kernel, and run all cells in order.

## Main Dependencies

```text
numpy
pandas
matplotlib
seaborn
scipy
scikit-learn
statsmodels
jupyter
```

## Current Limitations

- The same Validation period is used for Lasso alpha selection, Random Forest tuning, model-family selection, and risk-aversion selection.
- The residual covariance matrix is estimated from in-sample training residuals and remains fixed during the Test period.
- Transaction costs, bid–ask spreads, slippage, short-borrow costs, dividends, and financing costs are not included.
- Corporate actions are handled manually rather than through a comprehensive adjusted total-return dataset.
- The backtest should be audited further to confirm the exact signal, order, and execution timestamps.
- Target portfolio dollars are based on the initial-capital convention rather than dynamically compounding with current NAV.

Potential extensions include nested time-series cross-validation, rolling covariance estimation, covariance shrinkage, transaction-cost modeling, and portfolio-aware model-selection criteria.

## Key Takeaway

The main contribution of this project is not a single return number. It is a disciplined and reusable research loop connecting:

```text
Signal Engineering
→ Multi-Model Validation
→ Residual Risk Estimation
→ Constrained Optimization
→ Validation-Based Risk Selection
→ Final Test Evaluation
```

The project also illustrates an important practical lesson: a model with the lowest prediction MSE may not produce the best unoptimized portfolio, and careful risk construction can be as important as the prediction model itself.

## Disclaimer

This project is provided for educational and research purposes only. It does not constitute investment advice, a trading recommendation, or a claim of future performance.
