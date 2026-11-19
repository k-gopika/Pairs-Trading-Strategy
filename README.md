# Statistical Arbitrage: Pairs Trading Strategy in Financial Markets
An end-to-end quantitative research and algorithmic trading framework implementing a **market-neutral statistical arbitrage (pairs trading) strategy**. The strategy identifies cointegrated asset pairs across a multi-asset universe (equities and global indices), models mean-reverting price dynamics, generates dynamic z-score trading signals, and evaluates strategy performance with risk management constraints (stop-loss, leverage limits, and trade duration tracking).

---

## Table of Contents
- [Strategy Overview](#strategy-overview)
- [Mathematical & Statistical Framework](#mathematical--statistical-framework)
  - [Stationarity & Unit Root Testing](#stationarity--unit-root-testing)
  - [Engle-Granger Two-Step Cointegration](#engle-granger-two-step-cointegration)
  - [Spread vs. Price Ratio Modeling](#spread-vs-price-ratio-modeling)
  - [Dynamic Rolling Z-Score Formulation](#dynamic-rolling-z-score-formulation)
- [Asset Universe & Pair Screening](#asset-universe--pair-screening)
- [Execution Logic & Risk Management](#execution-logic--risk-management)
- [Backtest Performance & Key Metrics](#backtest-performance--key-metrics)
- [Repository Structure](#repository-structure)
- [Installation & Usage](#installation--usage)
- [Limitations & Future Enhancements](#limitations--future-enhancements)
- [Author](#author)

---

## Strategy Overview

Pairs trading is an investment strategy based on the mathematical concept of **cointegration**. Unlike correlation—which merely measures contemporaneous linear co-movement of asset returns—cointegration establishes a long-term structural equilibrium between non-stationary time series.

When two cointegrated assets temporarily diverge due to short-term supply/demand imbalances or market noise:
1. **Short** the relatively overvalued asset.
2. **Long** the relatively undervalued asset.
3. Close both positions when the spread/ratio mean-reverts to its equilibrium, locking in an arbitrage profit while remaining insulated from broad market directional risk (**market neutrality**).

```
   Asset Price
       ^
       |        /\      Short Asset A / Long Asset B (Z > +1.25)
   P_A |-------/--\---------------------------------------------
       |      /    \      Mean Reversion / Exit (|Z| < 0.5)
   P_B |-----/------\-------------------------------------------
       |    /        \/  Long Asset A / Short Asset B (Z < -1.25)
       +--------------------------------------------------------> Time
```

---

## Mathematical & Statistical Framework

### 1. Stationarity & Unit Root Testing
Financial asset prices $P_t$ are typically non-stationary processes integrated of order 1, denoted $P_t \sim I(1)$. A time series $Y_t$ is weak-sense stationary, $I(0)$, if:
$$\mathbb{E}[Y_t] = \mu \quad \forall t$$
$$\mathrm{Var}(Y_t) = \sigma^2 < \infty \quad \forall t$$
$$\mathrm{Cov}(Y_t, Y_{t-k}) = \gamma_k \quad \forall t, k$$

The **Augmented Dickey-Fuller (ADF)** test checks for the presence of a unit root:
$$\Delta Y_t = \alpha + \beta t + \gamma Y_{t-1} + \sum_{i=1}^p \delta_i \Delta Y_{t-i} + \varepsilon_t$$
- **Null Hypothesis ($H_0$)**: $\gamma = 0$ (Unit root present, non-stationary).
- **Alternative Hypothesis ($H_1$)**: $\gamma < 0$ (Stationary, mean-reverting).

### 2. Engle-Granger Two-Step Cointegration
Two $I(1)$ series $X_t$ and $Y_t$ are cointegrated, denoted $CI(1, 1)$, if there exists a cointegrating vector $[1, -\beta]$ such that a linear combination is stationary $I(0)$:
$$\varepsilon_t = Y_t - \beta X_t \sim I(0)$$

**Procedure:**
1. **Step 1 (OLS Regression)**: Estimate the static hedge ratio $\beta$:
   $$Y_t = \alpha + \beta X_t + \varepsilon_t$$
2. **Step 2 Residual Unit Root Test**: Perform an Augmented Dickey-Fuller (ADF) test on the estimated residuals, $\hat{\varepsilon}_t$:

$$
\Delta \hat{\varepsilon}_t =
\gamma \hat{\varepsilon}_{t-1}
+ \sum_{i=1}^{p} \delta_i \Delta \hat{\varepsilon}_{t-i}
+ u_t
$$
   *Note: Because residuals are estimated via OLS (minimizing variance), standard Dickey-Fuller critical values cannot be used. We use MacKinnon / Engle-Yoo augmented critical values.*

### 3. Spread vs. Price Ratio Modeling
The repository explores both the **OLS Residual Spread** and the **Price Ratio**:
- **Residual Spread**: $S_t = P_{A,t} - \beta P_{B,t}$ (Beta-neutral spread).
- **Price Ratio**: $R_t = \frac{P_{A,t}}{P_{B,t}}$ (Scale-free, naturally dollar-neutral, independent of price scale drift).

### 4. Dynamic Rolling Z-Score Formulation
To capture localized regime shifts without lookahead bias, signals are computed using rolling windows:
$$MA_{\text{fast}}(t) = \frac{1}{w_1} \sum_{k=0}^{w_1-1} R_{t-k}, \quad w_1 = 5$$
$$MA_{\text{slow}}(t) = \frac{1}{w_2} \sum_{k=0}^{w_2-1} R_{t-k}, \quad w_2 = 40$$
$$\sigma_{\text{slow}}(t) = \sqrt{\frac{1}{w_2 - 1} \sum_{k=0}^{w_2-1} \left(R_{t-k} - MA_{\text{slow}}(t)\right)^2}$$

The rolling Z-score is defined as:
$$Z_t = \frac{MA_{\text{fast}}(t) - MA_{\text{slow}}(t)}{\sigma_{\text{slow}}(t)}$$

---

## Asset Universe & Pair Screening

A multi-asset screening universe of **50 assets** across 5 years of daily adjusted close prices (2013-01-01 to 2018-01-01) was analyzed:
- **Major Banking Institutions (20)**: `JPM`, `BAC`, `WFC`, `C`, `GS`, `MS`, `DB`, `UBS`, `BBVA`, `SAN`, `ING`, `BNPQY`, `HSBC`, `SMFG`, `PNC`, `USB`, `BK`, `STT`, `KEY`, `RF`
- **Mega-Cap Technology Stocks (17)**: `AAPL`, `MSFT`, `GOOGL`, `AMZN`, `NVDA`, `TSLA`, `NFLX`, `ADBE`, `ORCL`, etc.
- **Global Macro Indices (13)**: `^DJI`, `^IXIC`, `^GSPC`, `^FTSE`, `^N225`, `^HSI`, `^VIX`, `^TNX`, etc.

Pairwise cointegration testing was evaluated over $\binom{50}{2} = 1,225$ combinations at significance threshold $\alpha = 0.02$.

### Selected Pair: Citigroup (`C`) & State Street Corporation (`STT`)
- **Economic Intuition**: Both are systemically important US banking and custodial financial institutions governed by identical macroeconomic drivers, Federal Reserve monetary policy, interest rate yield curves, and Basel III regulatory capital requirements.
- **Statistical Cointegration**: Full-sample Engle-Granger test confirms cointegration ($p < 0.02$).

---
## Risk Controls
1. **Capital Sizing**: Initial capital $V_0 = \$100,000$, split equally across both legs ($\$50,000$ per leg) to maintain dollar neutrality.
2. **Dynamic Entry**: Threshold $\pm 1.25\sigma$ filters out minor fluctuations and trades only statistically significant mispricings.
3. **Mean-Reversion Exit**: Position exits when $|Z_t| < 0.5\sigma$, locking in convergence gains before spread overshoots.
4. **Hard Stop-Loss**: Immediate liquidation if mark-to-market drawdown of an active trade exceeds $\$5,000$ to prevent divergence blowouts.

---

## Backtest Performance & Key Metrics

Simulated across 5-year daily data (2013-01-01 to 2018-01-01):

| Metric | Backtest Result | Description |
|---|---|---|
| **Initial Capital** | \$100,000 | Portfolio starting base |
| **Gross Profit** | \$402,698.71 | Total cumulative gains across all winning trades |
| **Gross Loss** | -\$23,355.20 | Total cumulative losses across losing trades |
| **Net Profit** | **\$379,343.52** | Net trading profit after stop-losses |
| **Cumulative Return** | **379.34%** | Total strategy return over 5 years |
| **Win Rate** | **89.66%** | Profitable trades percentage (26 / 29 closed cycles) |
| **Max Drawdown** | **-4.20%** | Peak-to-trough equity decline |
| **Average Winning Trade** | \$15,488.41 | Average profit per winning cycle |
| **Average Losing Trade** | -\$7,785.07 | Average loss per stopped-out cycle |
| **Largest Winning Trade** | \$37,455.83 | Maximum single-trade profit |
| **Largest Losing Trade** | -\$12,008.40 | Maximum single-trade loss |
| **Average Holding Duration**| 56 days | Typical trade holding time until mean reversion |

> **Quantitative Note on Sharpe Ratio:** The notebook reports an annualized Sharpe ratio of 14.68. As discussed in the interview defense documentation, this metric is computed across discrete trade-exit milestones rather than daily marked-to-market valuations. Marked-to-market daily accounting yields a realistic annualized Sharpe ratio of 1.5 - 2.2.

---

## Repository Structure

```
Pairs-Trading-Strategy/
├── Pair_Trading_Algorithm.ipynb    # Main research, screening, and simulation notebook
├── README.md                       # Quantitative strategy documentation
└── requirements.txt                # Python dependencies
```

---

## Installation & Usage

### Prerequisites
- Python 3.8+
- Jupyter Notebook / Google Colab

### Installation
```bash
# Clone the repository
git clone https://github.com/k-gopika/Pairs-Trading-Strategy.git
cd Pairs-Trading-Strategy

# Install required dependencies
pip install -r requirements.txt
```

### Dependencies
```text
numpy>=1.20.0
pandas>=1.3.0
statsmodels>=0.13.0
yfinance>=0.2.0
matplotlib>=3.4.0
seaborn>=0.11.0
scipy>=1.7.0
```

### Running the Notebook
```bash
jupyter notebook Pair_Trading_Algorithm.ipynb
```

---

## Limitations & Future Enhancements

1. **Dynamic Hedge Ratio via Kalman Filtering**:
   - *Current*: Fixed rolling ratio $R_t = P_A / P_B$.
   - *Enhancement*: Implement a State-Space model with Kalman filtering to dynamically update the cointegrating vector $[\alpha_t, \beta_t]$ in real-time.
2. **Ornstein-Uhlenbeck (OU) Parameterization**:
   - Model the spread as a continuous mean-reverting process: $dS_t = \theta(\mu - S_t)dt + \sigma dW_t$.
   - Solve for theoretical half-life $t_{1/2} = \frac{\ln 2}{\theta}$ and optimal analytical entry/exit boundaries.
3. **Execution Friction & Short Borrow Costs**:
   - Explicitly model borrow fees (locate fees for hard-to-borrow equities), bid-ask slippage, and execution latency.
4. **Out-of-Sample Walk-Forward Validation**:
   - Implement rolling walk-forward backtesting across out-of-sample regimes (e.g., 2018-2024, COVID crash, 2022 rate-hike cycle).

---

