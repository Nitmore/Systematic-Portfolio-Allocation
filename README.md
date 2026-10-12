# Systematic Portfolio Allocation

Mean-variance portfolio construction on Indian equities (NSE). The pipeline goes from a 102-stock daily return panel to a 12-stock, low-correlation book. It then traces a Markowitz efficient frontier with a fixed cash sleeve and transaction-cost accounting.

Built as an Investment Management course project (Sept–Oct 2025) in Google Colab.

---

## Pipeline

```
returns_2022_filled.csv  (869 days × 102 stocks, from 2022-01-03)
        │
        ▼
1. Load & clean ─────────── % → decimal auto-fix, drop near-constant columns
        │
        ▼
2. Trading-day panel ────── drop all-zero rows (867 kept), infer annualisation
        │                    factor from the date span (≈ 247 obs/year)
        ▼
3. Expected returns ─────── arithmetic & geometric annualised ER per stock
        │
        ▼
4. Positive-ER filter ───── keep stocks with geometric annual ER > 0  → 91 stocks
        │
        ▼
5. Covariance / correlation (daily & annualised)
        │
        ▼
6. Low-correlation selection ─ greedy pick of 12 stocks minimising mean |ρ|
        │
        ▼
7. Optimisation ─────────── efficient frontier, min-variance, target-vol
                             (5% cash, 0.1% TC, long-only or capped shorts)
```

### Stock selection (step 6)

This is a greedy, farthest-first style heuristic on the absolute correlation matrix:

1. Seed with the stock that has the lowest average |ρ| to all others.
2. Repeatedly add the stock with the lowest mean |ρ| to the stocks already selected.
3. Stop at *k* = 12.

Using |ρ| penalises strong negative and strong positive co-movement equally. The result is a heuristic, not a global optimum.

### Optimisation (step 7)

The decision variable is `x`, the weights within the risky sleeve, with `Σx = 1`. Total portfolio weights are `w = 0.95·x`, with 5% held in cash.

| Problem | Objective | Constraints |
|---|---|---|
| Efficient frontier (60 points) | min `xᵀΣx` | `Σx = 1`, `μᵀx = target`, `x ≥ 0` |
| Minimum variance | min `xᵀΣx` | `Σx = 1`, `x ≥ 0` (or `‖x‖₁ ≤ 1.8` with shorts) |
| Target volatility | max `μᵀx` | `Σx = 1`, `xᵀΣx ≤ (σ*/0.95)²` |

- **Covariance** is the sample covariance, annualised, then projected to the nearest PSD matrix by eigenvalue clipping.
- **Transaction cost** is 0.1% of gross risky exposure (`Σ|w|`), deducted from the cash sleeve. It is applied after optimisation, as a cost/reporting adjustment, and is not a term in the objective.
- **Solvers** are tried in order: ECOS, then OSQP, then SCS.

---

## Results (in-sample)

Selected 12-stock book:

`JYOTHYLAB, CIPLA, CRAFTSMAN, SAPPHIRE, MAXHEALTH, KPRMILL, POWERINDIA, APARINDS, PHOENIXLTD, VIJAYA, ITC, SUVENPHAR`

Annualised expected returns range from 15.3% (SAPPHIRE) to 82.4% (APARINDS).

**Minimum-variance portfolio (long-only, 5% cash, capital ₹1 crore):**

| Metric | Value |
|---|---|
| Expected annual return | 24.25% |
| Annualised volatility | 12.26% |
| Return / vol (rf = 0) | 1.98 |
| Transaction cost | ₹9,500 (0.095%) |
| Cash after TC | 4.91% |

| Stock | Weight | Stock | Weight |
|---|---|---|---|
| ITC | 30.2% | CRAFTSMAN | 5.0% |
| CIPLA | 20.4% | MAXHEALTH | 4.7% |
| SAPPHIRE | 7.7% | PHOENIXLTD | 4.2% |
| JYOTHYLAB | 7.1% | VIJAYA | 1.7% |
| KPRMILL | 6.2% | POWERINDIA | 1.2% |
| SUVENPHAR | 5.7% | APARINDS | 0.9% |

**Target-volatility run:** a 1% volatility target is infeasible. With cash fixed at 5%, the minimum achievable volatility is 12.26%, both long-only and with shorts capped at 1.8× gross. Allowing shorts did not lower it, because the unconstrained min-variance solution is already all-long.

---

## Repository contents

| File | Description |
|---|---|
| `IM_portfolio.ipynb` | Full pipeline notebook (Colab) |

### Generated outputs (written next to the input file)

| File | Contents |
|---|---|
| `expected_returns.csv` | Daily mean, annual arithmetic & geometric ER per stock |
| `returns_positive.csv` | Return panel after positive-ER filter |
| `covariance_daily.csv`, `covariance_annual.csv`, `correlation.csv` | Risk matrices for the filtered universe |
| `selected_12_stocks.csv` | Chosen tickers |
| `portfolio_12_returns.csv` | Daily returns of the 12-stock book |
| `efficient_frontier_weights_with_TC.csv` | 60 frontier points: return, vol, TC, cash, per-stock weights |
| `target_vol_*_weights_with_TC*.csv` | Target-vol weights (written only when feasible) |

---

## How to run

1. Open `IM_portfolio.ipynb` in Google Colab.
2. Put `returns_2022_filled.csv` in your Drive. Its first column should be `Date`, followed by one column of daily returns per ticker. Returns can be decimals or percentages.
3. Edit `file_path` in cell 2 if your path differs.
4. Run all cells in order. The optimisation cells install `cvxpy`, `ecos` and `osqp` automatically.

### Key configuration

| Parameter | Default | Meaning |
|---|---|---|
| `FILTER_METRIC` | `"geo_annual"` | ER measure for the positive-return filter |
| `K` | `12` | Number of stocks to select |
| `USE_ABS` | `True` | Use \|ρ\| in selection |
| `CASH_W` | `0.05` | Fixed cash weight |
| `RISK_FREE` | `0.00` | Annual cash yield |
| `GRID_N` | `60` | Frontier points |
| `CAPITAL_RS` | `1e7` | Capital (₹) for TC in rupees |
| `ALLOW_SHORTS` / `L1_GROSS_LIMIT` | `False` / `None` | Shorting and gross-exposure cap |
| `SIGMA_TARGET` | `0.01` | Target annual volatility |

**Dependencies:** `pandas`, `numpy`, `cvxpy`, `ecos`, `osqp`, `matplotlib`, `openpyxl`

---

## Limitations

- **In-sample only.** Expected returns, the positive-ER filter, stock selection and optimisation all use the same 2022–25 window. There is no backtest or out-of-sample evaluation, so the reported returns are estimates, not realised performance.
- **Raw historical means as ER.** Sample means are noisy, and the very high values (up to 82%) drive the frontier. No shrinkage or Black-Litterman adjustment is applied.
- **Transaction costs do not enter the objective.** They are a fixed deduction, which for long-only portfolios is always ₹9,500.
- **Risk-free rate is 0.** Return/vol ratios are therefore overstated relative to Indian T-bill yields.
- **Static allocation.** There is no rebalancing schedule or turnover modelling.

## Possible extensions

- Add a turnover penalty `λ‖w − w_prev‖₁` inside the objective.
- Run a walk-forward backtest with monthly or quarterly rebalancing.
- Use Ledoit-Wolf covariance shrinkage and shrunk expected returns.
- Add a maximum-Sharpe portfolio with a realistic risk-free rate.
- Add sector and position-size caps.
