# Quant Backtesting — ASX Strategy Library & Screener

A full quantitative trading research project built from scratch in Python.  
Five individual trading strategies each backtested across 6 global assets, then combined into an automated multi-strategy screener that runs across 25 ASX stocks with daily output.

> **Core finding:** Strategy-asset fit matters more than the strategy itself. The same strategy that produces a Sharpe of 0.87 on one stock can produce -0.21 on another. The screener exists to find which combination actually works.

---

## Project Structure

```
├── strategies/
│   ├── 01_moving_average/       — MA Crossover: trend-following baseline
│   ├── 02_rsi/                  — RSI: momentum extremes mean reversion
│   ├── 03_bollinger_bands/      — Bollinger Bands: volatility band reversion
│   ├── 04_macd/                 — MACD: hybrid momentum + trend
│   └── 05_mean_reversion/       — Mean Reversion: Z-score statistical reversion
├── screener/
│   ├── Goofy Screener v2.ipynb  — Interactive notebook (main display)
│   └── goofy_screener_daily.py  — Automated daily script
└── README.md
```

---

## Methodology

All strategies use the same validation framework throughout:

| | Detail |
|---|---|
| **Train period** | Jan 2016 → Dec 2020 (in-sample) |
| **Test period** | Jan 2021 → present (out-of-sample) |
| **Parameter selection** | Grid search on training data only |
| **Signal execution** | `.shift(1)` on all signals — no lookahead bias |
| **Returns** | Log returns, annualised |
| **Sharpe ratio** | Annualised return ÷ annualised volatility |
| **Max drawdown** | Peak-to-trough on cumulative equity curve |
| **Benchmark** | Buy & Hold (passive holding of the same asset) |

The train/test split is the core of the methodology. Parameters are chosen purely on historical data, then locked and applied to future data the model never saw. This prevents overfitting and gives a genuine signal on whether a strategy generalises.

---

## Strategy 1 — Moving Average Crossover

**File:** `strategies/01_moving_average/Goofy MA for 6 assets.ipynb`

**Logic:** Goes long when the fast MA crosses above the slow MA; exits when it crosses below. Pure trend-following — rides momentum, avoids counter-trend moves.

**Parameters tested:** 4 MA pairs × 6 assets = 24 combinations  
`(10/30), (20/50), (25/60), (30/70)`

**Assets:** NVDA, SPY, CBA.AX, BHP.AX, 7203.T (Toyota), 6758.T (Sony)

### Key Findings

| Asset | Best Pair | IN Sharpe | OUT Sharpe | Beats B&H |
|---|---|---|---|---|
| NVDA | MA20/50 | High | Moderate | No — AI bull run too strong |
| SPY | MA10/30 | Moderate | Moderate | Closest to beating B&H of any strategy |
| CBA.AX | MA20/50 | Moderate | Moderate | No |
| BHP.AX | MA25/60 | Low | Low | No |
| 7203.T | MA10/30 | Low | Low | No |
| 6758.T | MA20/50 | Low | Low | No |

**Lesson:** MA crossover is the cleanest trend-following tool but the 2021–2026 bull market meant being in cash during corrections was costly. SPY was the best fit — broad index with smoother trends than individual stocks.

---

## Strategy 2 — RSI (Relative Strength Index)

**File:** `strategies/02_rsi/Goofy8 RSI for 6 assets.ipynb`

**Logic:** Buys when RSI falls below the oversold threshold (stock has been beaten down); sells when RSI rises above the overbought threshold (stock has run up). Captures mean reversion through momentum extremes.

**Parameters tested:** 4 RSI periods × 4 threshold pairs × 6 assets = 96 combinations  
Periods: `7, 9, 14, 21` | Thresholds: `(70/30), (65/35), (75/25), (80/20)`

**Assets:** NVDA, SPY, CBA.AX, BHP.AX, 7203.T (Toyota), 6758.T (Sony)

### Key Findings

| Asset | Best Params | IN Sharpe | OUT Sharpe | Notable |
|---|---|---|---|---|
| CBA.AX | RSI14 (30/70) | Stable | Stable | Barely decayed out-of-sample — most consistent result |
| SPY | RSI14 (35/75) | Moderate | Low | Efficient market, hard to exploit |
| NVDA | RSI21 (25/65) | Low | Very low | Trend too strong for mean reversion signals |
| BHP.AX | RSI14 (30/70) | Low | Low | Commodity cycles disrupt RSI signals |

**Standout result — CBA.AX:** RSI Sharpe barely decayed from in-sample to out-of-sample. This is rare and suggests a genuine fit between RSI and a trending bank stock — CBA oscillates predictably enough for RSI to add value.

**Lesson:** RSI works best when an asset regularly overshoots and snaps back. Strong trending assets like NVDA punish RSI — it kept generating sell signals during an AI-driven multi-year uptrend.

---

## Strategy 3 — Bollinger Bands

**File:** `strategies/03_bollinger_bands/Goofy BB for 6 assets.ipynb`

**Logic:** Buys when price touches or breaks below the lower band (2 standard deviations below the 20-day SMA); sells when price returns to the upper band. Treats extreme deviations from the rolling mean as statistically likely to revert.

**Parameters tested:** 4 windows × 4 std dev widths × 6 assets = 96 combinations  
Windows: `10, 15, 20, 30` | StdDev: `1.5, 2.0, 2.5, 3.0`

**Assets:** NVDA, SPY, CBA.AX, BHP.AX, 7203.T (Toyota), 6758.T (Sony)

### Key Findings

| Asset | Best Params | IN Sharpe | OUT Sharpe | Beats B&H |
|---|---|---|---|---|
| 6758.T (Sony) | BB20 (2.0σ) | 0.91 | **0.87** | **Yes** |
| CBA.AX | BB20 (2.0σ) | 0.65 | 0.58 | No |
| SPY | BB15 (1.5σ) | 0.52 | 0.41 | No |
| NVDA | BB10 (1.5σ) | 0.29 | -0.18 | No |
| BHP.AX | BB20 (2.0σ) | 0.31 | 0.22 | No |
| 7203.T | BB20 (2.5σ) | 0.44 | 0.28 | No |

**Standout result — Sony (6758.T):** The only asset across all 5 strategies to beat Buy & Hold out-of-sample. Sony's price structurally oscillates between support and resistance ranges — it is genuinely mean-reverting. Sharpe held at 0.87 out-of-sample, which is strong. This validates the core thesis: the right strategy on the right asset produces real edge.

**Lesson:** Bollinger Bands needs a range-bound asset to work. It completely failed on NVDA (negative Sharpe out-of-sample) because a parabolic trend never reverts to a 20-day average. Japanese stocks and ASX bank stocks were better fits than US tech.

---

## Strategy 4 — MACD (Moving Average Convergence Divergence)

**File:** `strategies/04_macd/Goofy MACD for 6 assets.ipynb`

**Logic:** Goes long when the MACD line crosses above its signal line; exits when it crosses below. A hybrid strategy — uses exponential moving averages to capture both trend direction and momentum shifts.

**Parameters tested:** 2 fast periods × 2 slow periods × 2 signal periods × 6 assets = 48 combinations  
Fast: `8, 12` | Slow: `21, 26` | Signal: `7, 9`

**Assets:** NVDA, SPY, CBA.AX, BHP.AX, 7203.T (Toyota), 6758.T (Sony)

### Key Findings

| Asset | Best Params | IN Sharpe | OUT Sharpe | Notable |
|---|---|---|---|---|
| NVDA | MACD(8,21,7) | 0.94 | **1.03** | Sharpe *improved* out-of-sample — rare |
| CBA.AX | MACD(12,26,9) | 0.26 | **0.73** | Tripled out-of-sample |
| SPY | MACD(8,21,9) | 0.55 | 0.48 | Moderate, consistent |
| 6758.T (Sony) | MACD(12,26,9) | 0.44 | 0.31 | Failed — Sony is mean-reverting, not trend |
| BHP.AX | MACD(12,26,9) | 0.28 | 0.19 | Weak across all params |
| 7203.T | MACD(8,26,7) | 0.38 | 0.24 | Inconclusive |

**Standout result — NVDA:** The only strategy-asset combination where out-of-sample Sharpe exceeded in-sample (0.94 → 1.03). This is extremely unusual and suggests MACD captures something real about NVDA's momentum structure. The AI-driven trend accelerated the signals that were already working.

**CBA surprise:** MACD Sharpe tripled from in-sample to out-of-sample (0.26 → 0.73). CBA's post-2021 trending behaviour suited MACD's momentum signals better than the range-bound 2016–2020 training period.

**Sony contrast:** MACD failed on Sony (Sharpe 0.31) while BB produced 0.87 on the same asset. This is the clearest demonstration in the project of why strategy-asset fit matters — the strategy changed, the asset didn't.

**Lesson:** MACD is the best tool for sustained trending assets. The EMA construction means it picks up trend shifts earlier than simple MA crossover, but it will underperform on mean-reverting assets like Sony.

---

## Strategy 5 — Mean Reversion (Z-Score)

**File:** `strategies/05_mean_reversion/Goofy Mean Reversion for 6 assets.ipynb`

**Logic:** Goes long when an asset's price falls more than N standard deviations below its rolling mean (Z-score < -threshold); exits when price returns to the mean (Z-score ≥ 0). Pure statistical mean reversion with no momentum component.

**Parameters tested:** 3 windows × 3 thresholds × 6 assets = 54 combinations  
Windows: `10, 20, 40` | Thresholds: `1.0, 1.5, 2.0`

**Assets:** NVDA, SPY, CBA.AX, BHP.AX, 7203.T (Toyota), 6758.T (Sony)

### Key Findings

| Asset | Best Params | IN Sharpe | OUT Sharpe | Beats B&H |
|---|---|---|---|---|
| 6758.T (Sony) | Z(20, 1.5) | **1.34** | 0.41 | No |
| SPY | Z(20, 1.5) | 0.58 | 0.51 | No |
| CBA.AX | Z(10, 1.0) | 0.47 | 0.43 | No |
| BHP.AX | Z(20, 2.0) | 0.31 | 0.18 | No |
| NVDA | Z(10, 1.5) | 0.22 | -0.31 | No |
| 7203.T | Z(20, 1.5) | 0.44 | 0.19 | No |

**Critical finding — regime dependence:** No asset beat Buy & Hold out-of-sample. This is not a failure of the strategy — it is a regime problem. 2021–2026 was a sustained bull market. Pure mean reversion only works in sideways or range-bound markets where price genuinely oscillates around a stable mean.

**Sony warning sign:** Sony had the highest in-sample Sharpe of any asset across all five strategies (1.34). Out-of-sample it fell to 0.41. Classic overfitting signal — parameters perfectly calibrated to Sony's 2016–2020 behaviour, but that regime shifted post-2021.

**SPY and CBA held up:** Both maintained their Sharpe from in-sample to out-of-sample. For both, the strategy provided genuine downside protection even without beating B&H on raw return.

**Lesson:** Before applying mean reversion, identify whether the asset is currently in a trending or range-bound regime. Applying this strategy in a bull market is the wrong tool for the environment.

---

## Phase 2 — Automated Multi-Strategy Screener

**Files:**  
`screener/Goofy Screener v2.ipynb` — full interactive notebook with charts  
`screener/goofy_screener_daily.py` — automated daily script (schedulable)

### What it does

The screener automates everything built across the 5 individual strategies into a single pipeline:

1. Downloads live price data for 25 ASX assets via Yahoo Finance
2. Splits into training (2016–2020) and test (2021–present) periods
3. Runs a full grid search across all 5 strategies and their parameter combinations on training data only
4. Selects the best strategy per asset by in-sample Sharpe ratio
5. Applies the chosen strategy to out-of-sample test data — data the selection process never touched
6. Reports: Sharpe, total return, max drawdown, B&H comparison, and robustness score
7. Exports a colour-coded Excel report and equity curve charts

### ASX Universe (25 assets)

| Sector | Tickers |
|---|---|
| Banks | CBA.AX, WBC.AX, ANZ.AX, NAB.AX |
| Mining | BHP.AX, RIO.AX, FMG.AX, S32.AX, MIN.AX |
| Energy | WDS.AX, STO.AX, BPT.AX |
| Healthcare | CSL.AX, RMD.AX, COH.AX, SHL.AX |
| Retail | WES.AX, WOW.AX, COL.AX |
| Tech | XRO.AX, CPU.AX, WTC.AX |
| Infrastructure | TCL.AX, APA.AX, AGL.AX |
| ETFs | IOZ.AX, STW.AX, VAS.AX |

### Screener Results (Run: 2026-04-12)

**Overall:** 25 assets screened | 4/25 beat Buy & Hold | 22/25 High Robustness

**Strategy distribution across the ASX universe:**

| Strategy | Count |
|---|---|
| MA Crossover | 8 |
| RSI | 6 |
| MACD | 5 |
| Mean Reversion | 3 |
| Bollinger Bands | 3 |

**Top 5 by Out-of-Sample Sharpe Ratio:**

| Asset | Strategy | Robustness | OUT Sharpe | OUT Return % | OUT Max DD % | Beats B&H |
|---|---|---|---|---|---|---|
| CBA.AX | RSI | ✓ High | **0.993** | +52.7% | -11.0% | No |
| NAB.AX | MACD | ✓ High | **0.923** | +81.7% | -12.1% | No |
| RIO.AX | RSI | ✓ High | **0.865** | +93.3% | -22.6% | No |
| STW.AX | RSI | ✓ High | **0.842** | +49.4% | -13.0% | No |
| ANZ.AX | MACD | ✓ High | **0.692** | +59.8% | -16.1% | No |

### Reading the results

**Why only 4/25 beat Buy & Hold — and why that's the honest result:**  
2021–2026 was a strong bull market on the ASX. In sustained uptrends, passive holding tends to win on raw return because rule-based strategies go to cash during corrections and miss some of the recovery. This is not a flaw in the screener — it is an accurate reflection of what quantitative strategies do in trending markets.

**What the strategies actually provide:**  
The real value is drawdown protection. The top 5 assets show strategy max drawdowns of -11% to -23% during the test period. The same assets held passively experienced drawdowns of -35% to -55% during that same period. For investors who cannot tolerate large paper losses, a strategy that earns less but limits the worst drawdown to -15% is genuinely useful.

**Robustness score explained:**  
For each selected strategy, the surrounding parameter space is tested. If similar parameters produce similar Sharpe ratios → ✓ High (the result is stable, not a lucky single point). If the result only holds for one exact parameter combination → ⚠ Low (likely overfitted). 22/25 High robustness across the universe is a strong result.

---

## Honest Limitations

- **No transaction costs.** All trades are assumed free. Real brokerage and slippage would reduce returns, particularly for strategies that signal frequently.
- **Sharpe without risk-free rate.** Calculated as `annualised return / annualised volatility`. Subtracting the Australian cash rate (~4%) would reduce headline Sharpe numbers by approximately 0.3–0.5.
- **Multiple comparisons.** Selecting the best of 5 strategies × multiple parameter sets introduces selection bias even with a proper train/test split. Out-of-sample numbers are slightly optimistic estimates of true generalisation performance.
- **Long-only.** All strategies are long or flat (0 or 1 position). No short selling.

---

## How to Run

### Requirements
```
pip install yfinance pandas numpy matplotlib openpyxl
```

### Individual strategy notebooks
Open any notebook in Jupyter or Anaconda and run all cells. Each notebook is fully self-contained.

### Screener — interactive
Open `screener/Goofy Screener v2.ipynb` and run all cells.  
Output: equity curve charts + colour-coded Excel report saved to `screener_output/`

### Screener — automated
```bash
python screener/goofy_screener_daily.py
```
Change `UNIVERSE = "ASX"` to `"US"` or `"CUSTOM"` in the config section to run on a different asset list.

---

## Key Takeaways

**1. Strategy-asset fit is everything.**  
Sony beat Buy & Hold with Bollinger Bands (Sharpe 0.87) but completely failed with MACD (Sharpe 0.31). The strategy didn't change — the asset structure did.

**2. Out-of-sample validation is non-negotiable.**  
Sony's Mean Reversion in-sample Sharpe was 1.34 — the highest single result in the entire project. Out-of-sample it fell to 0.41. Without the train/test split, that would look like the best strategy in the study.

**3. Sharpe ratio matters more than raw return.**  
CBA's RSI strategy returned +52.7% with a Sharpe of 0.99 and max drawdown of -11%. Buy & Hold returned +167% but with a drawdown exceeding -35%. Which is better depends entirely on how much risk you can bear.

**4. Regime determines strategy performance.**  
Mean reversion underperformed across the board in 2021–2026 because the market was trending. The same strategies applied in a sideways 2015–2016 environment would tell a completely different story.

**5. Robustness separates signal from luck.**  
A result that only appears at one exact parameter setting is almost certainly noise. A result that holds across a wide parameter neighbourhood is more likely to reflect a genuine structural property of the asset.

---

## Disclaimer

This project is for educational and research purposes only. All backtested results are historical simulations and do not guarantee future performance. Nothing in this repository constitutes financial advice.

---

*Built by Hiroki Kunu — International Finance, University of Queensland*  
*GitHub: [GoofyisDAWG](https://github.com/GoofyisDAWG) | LinkedIn: [Hiroki Kunu](https://www.linkedin.com/in/hiroki-kunu-ba4218401)*
