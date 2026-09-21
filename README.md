# Finance Projects — Event Study and Optimal Trade Design

This repository contains three connected quantitative-finance notebooks. Together, they move from a **general position-sizing framework**, to an **empirical event study of Interactive Brokers (IBKR)**, and finally to a **retrospective trade-design application** that converts the measured event residual into an implementable hedged position.

The overall research pipeline is:

```math
\text{Event study}
\rightarrow
\text{counterfactual return}
\rightarrow
\text{abnormal return}
\rightarrow
\text{tradable residual}
\rightarrow
\text{execution costs and liquidity}
\rightarrow
\text{constrained position } q^*
```

The project is educational and retrospective. It is **not a live trading recommendation**.

---

## Repository structure

```text
ibkr_pca_analysis/
│
├── README.md
├── optimal-trade-design(1).ipynb
├── ibkr-pca-event-study.ipynb
└── ibkr-trading-design.ipynb
```

When executed, the notebooks may also use or create local folders such as:

```text
data/
├── raw/
└── derived/

output/
```

The integrated trading notebook expects a validated event-study handoff under:

```text
data/derived/ibkr_event_handoff/
```

with files such as:

```text
manifest.json
models.npz
event_outputs.parquet
model_audit.parquet
market_data_handoff.parquet
```

The trading-design notebook is designed to stop if the required handoff is missing or inconsistent with the frozen event-study source.

---

# 1. `optimal-trade-design(1).ipynb`

## Purpose

This notebook develops a **general, model-agnostic framework for translating a return distribution into a signed optimal position**.

It does not assume that returns come from any specific forecasting model. The input can be a bootstrap distribution, Bayesian posterior, event-study distribution, or another set of predictive return draws.

The decision variable is a dollar position `q`, where:

- `q > 0` means long;
- `q < 0` means short;
- `q = 0` means no trade.

The notebook separates three types of inputs:

1. **Model inputs** — the predictive return distribution;
2. **Market inputs** — transaction costs, market impact and liquidity;
3. **Investor-policy inputs** — wealth, risk aversion, loss limits and participation limits.

## Main methodology

Implementation costs are represented as:

```math
C(q)
=
c|q|
+
\frac{1}{2}\kappa q^2
```

where:

- `c|q|` captures proportional trading costs;
- `0.5 κ q²` captures nonlinear market impact.

Using a local CRRA certainty-equivalent approximation, the signed position is chosen by balancing expected return against costs, market impact and risk:

```math
J(q)
=
q\mu
-
c|q|
-
\frac12
\left(
\kappa
+
\frac{\rho}{W}\sigma^2
\right)q^2
```

The notebook then imposes:

- an Expected Shortfall loss limit;
- a market-capacity constraint;
- explicit long / short / no-trade comparison;
- uncertainty propagation across plausible input states.

The final object can therefore be a distribution of optimal positions rather than a single point estimate:

```math
q^{*(1)},\ldots,q^{*(B)}
```

## Role in the repository

This notebook is the **general theoretical trade-sizing framework** used later by the IBKR application. It can also be used independently with another event or return model.

---

# 2. `ibkr-pca-event-study.ipynb`

## Research question

This notebook studies Interactive Brokers around its migration from the **S&P MidCap 400 to the S&P 500**, announced after the close on 25 August 2025 and effective before the open on 28 August 2025.

The central question is:

> How much of IBKR's return around the index migration can be explained by common movements in economically related stocks, and how much remains as an event-associated abnormal return?

Because IBKR simultaneously entered the S&P 500 and left the S&P MidCap 400, the event is treated as a **net index migration**, not as a pure S&P 500 inclusion.

## Counterfactual model

The notebook builds a donor universe of 11 brokerage, exchange and market-infrastructure stocks:

```text
SCHW, HOOD, LPLA, RJF, CME, CBOE, NDAQ, ICE, MKTX, TW, VIRT
```

SPY is used as a benchmark rather than as a PCA donor.

The donor returns are standardized and compressed using PCA. IBKR is then regressed on the retained principal components:

```math
r_t^{IBKR}
=
\alpha
+
\beta_1 PC_{1,t}
+\cdots+
\beta_K PC_{K,t}
+
\varepsilon_t
```

The counterfactual return is the fitted value:

```math
\widehat r_t^{IBKR}
```

and the abnormal return is:

```math
AR_t
=
r_t^{IBKR}
-
\widehat r_t^{IBKR}
```

## Model selection and validation

The number of components is selected using **chronological walk-forward validation**, rather than an arbitrary explained-variance threshold.

The development sample compares `K = 1, ..., 11`. A moving-block-bootstrap version of the **one-standard-error rule** selects the simplest model whose RMSE is sufficiently close to the empirical minimum.

The final specification uses:

```math
K=2
```

for both close-to-close and open-to-close models.

These two components explain approximately:

- **53.5%** of donor variation for close-to-close returns;
- **53.1%** for open-to-close returns.

On the untouched 63-session holdout:

| Metric | Close-to-close PCR | Open-to-close PCR |
|---|---:|---:|
| RMSE | 171.4 bps | 122.1 bps |
| MAE | 116.5 bps | 87.0 bps |
| Out-of-sample R² | 0.712 | 0.789 |
| Actual-predicted correlation | 0.894 | 0.932 |

## Main empirical result

The primary tradeable window is:

```text
26 August 2025 open
→
27 August 2025 close
```

For this window, the estimated abnormal simple return is approximately:

```math
-5.24\%
```

The bootstrap median event effect is approximately **−5.93%**, with a pointwise 95% interval of approximately:

```math
[-10.24\%,-1.95\%]
```

Inference is limited by the small holdout sample. The primary-tradeable empirical placebo p-value is approximately **0.032**, but after Holm correction across the four related event windows, the adjusted p-value is approximately **0.125**.

The result is therefore economically meaningful and stable across several robustness checks, but it is **not statistically significant at the family-wide 5% level**.

## Robustness checks

The notebook includes:

- chronological holdout validation;
- historical residual placebo windows;
- Holm multiple-testing adjustment;
- Bonferroni-adjusted intervals;
- leave-one-donor-out estimation;
- winsorized-training sensitivity;
- residual bias and serial-correlation diagnostics;
- timing and volume analysis.

Removing any one donor leaves the primary-tradeable estimate roughly between **−5.44% and −5.03%**, while winsorization changes it only from approximately **−5.24% to −5.10%**.

## Interpretation

The PCA/PCR model is an **ex-post counterfactual**, not an ex-ante forecast and not a causal identification strategy.

The residual can contain:

- passive index demand;
- active investor repositioning;
- anticipation and arbitrage;
- IBKR-specific information;
- omitted common factors;
- model error.

The notebook therefore interprets the result as an **event-associated abnormal return**, rather than claiming that passive funds caused the full observed residual.

---

# 3. `ibkr-trading-design.ipynb`

## Purpose

This notebook connects the event-study project with the general trade-design framework.

It asks:

> Conditional on the event-return distribution estimated ex post, what position would the trade-design framework imply after accounting for hedge construction, trading costs, liquidity, risk and investor constraints?

The notebook does **not** claim that the −5.24% event residual was known before the event.

Instead, it is a retrospective exercise showing how a measured statistical opportunity can be translated into a theoretical trade.

## From statistical abnormal return to tradable residual

The event-study model produces:

```math
AR
=
r_{IBKR}
-
\widehat r_{IBKR}
```

The frozen PCR counterfactual is algebraically rewritten in the original donor-stock space:

```math
\widehat r_{IBKR}
=
a+w^\top r_{donors}
```

Because the regression intercept `a` is not directly tradable, the notebook defines the approximate tradable residual as:

```math
R^{res}
=
r_{IBKR}
-
w^\top r_{donors}
```

Therefore:

```math
R^{res}
=
AR+a
```

The primary-tradeable statistical abnormal return is approximately **−5.24%**, while the corresponding approximate tradable residual is approximately:

```math
-5.12\%
```

The notebook derives separate donor exposures for the open-to-close and close-to-close legs and explicitly models the required hedge rebalance between them.

## Trading costs and liquidity

The implementation model includes:

- daily dollar-volume liquidity;
- Corwin-Schultz bid-ask-spread proxies;
- commission assumptions;
- stock-borrow assumptions;
- square-root market-impact calibration;
- a local quadratic impact approximation;
- per-security, per-execution capacity limits.

For every security `s` and execution event `e`, capacity is checked as:

```math
|q||m_{s,e}|
\le
\phi V_s
```

Hence:

```math
q_{capacity}
=
\min_{s,e}
\frac{\phi V_s}{|m_{s,e}|}
```

This makes capacity a **multi-leg weakest-link constraint**, rather than a limit based only on IBKR liquidity.

## Educational investor mandate

The final application uses the following policy choices:

- portfolio wealth: **\$50 million**;
- CRRA risk aversion: **4**;
- Expected Shortfall loss budget: **\$500,000**;
- ES tail probability: **5%**;
- maximum participation per execution: **5% of dollar volume**.

These are investor-policy assumptions, not estimated market parameters.

## Main trade-design result

Under the central calibration, the optimizer selects approximately:

```math
q^*
=
-\$13.42\text{ million}
```

of IBKR target notional, equivalent to roughly **209,000 IBKR shares short**, together with the dynamically rebalanced donor hedge.

At this position:

- expected net P&L is approximately **\$540k**;
- modelled P&L volatility is approximately **\$249k**;
- 5% Expected Shortfall is approximately **−\$65.8k**.

The ES loss budget is therefore not close to binding. The main constraint is execution capacity.

Across 1,000 uncertainty iterations:

- median `q*`: approximately **−\$13.42m**;
- 5th–95th percentile range: approximately **−\$15.76m to −\$12.80m**;
- short direction selected: **100%**;
- capacity binding: **99.1%**;
- unconstrained risk optimum binding: **0.5%**;
- Expected Shortfall binding: **0.4%**.

The narrow position range should not be interpreted as precise knowledge of the unconstrained economic optimum. Because capacity binds almost all the time, it mainly reflects uncertainty in **feasible execution capacity**.

---

# How the three notebooks fit together

| Notebook | Main question | Main output |
|---|---|---|
| `optimal-trade-design(1).ipynb` | Given a return distribution, costs and constraints, how large should a signed position be? | Generic optimizer for `q*` |
| `ibkr-pca-event-study.ipynb` | How unusual was IBKR's return around its S&P 400 → S&P 500 migration? | Ex-post abnormal-return distribution |
| `ibkr-trading-design.ipynb` | How can that residual be hedged and translated into a constrained theoretical trade? | Tradable residual, execution model and IBKR target position |

A useful way to read the repository is:

```math
\text{Event study}
\rightarrow
\text{Trade-design theory}
\rightarrow
\text{Integrated IBKR application}
```

---

# Running the project

## 1. Clone the repository

```bash
git clone <your-repository-url>
cd finance_projects
```

## 2. Create an environment

The notebooks rely primarily on:

```text
numpy
pandas
scipy
matplotlib
scikit-learn
statsmodels
pandas-market-calendars
yfinance
seaborn
pyarrow
jupyter
```

For example:

```bash
python -m venv .venv
source .venv/bin/activate
pip install jupyter numpy pandas scipy matplotlib scikit-learn statsmodels pandas-market-calendars yfinance seaborn pyarrow
```

The event-study notebook also checks for its required packages and installs missing ones.

## 3. Recommended execution order

### A. Event study

Run:

```text
ibkr-pca-event-study.ipynb
```

This downloads/caches Yahoo Finance data, builds the donor-return panel, validates the PCA/PCR counterfactual and estimates the event residual.

Internet access is required on the first run unless a valid market-data cache is already present.

### B. Generic optimizer

Run:

```text
optimal-trade-design(1).ipynb
```

This notebook is independent of IBKR and develops the generic signed position-sizing framework.

### C. Integrated IBKR trade design

Run:

```text
ibkr-trading-design.ipynb
```

This notebook requires the validated event-study handoff in:

```text
data/derived/ibkr_event_handoff/
```

It verifies the required artifacts and the frozen source notebook before using the event-study outputs.

If the handoff files are absent, the notebook stops rather than silently reconstructing or approximating the frozen model.

---

# Data and reproducibility

Market data are obtained from **Yahoo Finance**. The event-study notebook caches downloaded data locally because historical vendor data may be revised.

The research design attempts to avoid information leakage by:

- separating development and holdout periods chronologically;
- excluding the possible-anticipation period from model fitting;
- selecting model complexity before observing the event;
- freezing the PCA/PCR model before calculating event abnormal returns;
- validating downstream handoff artifacts.

Random seeds are set inside the notebooks for reproducible bootstrap and Monte Carlo calculations.

---

# Main limitations

The main limitations of the repository are deliberate and important:

1. **The event study is ex post.** Contemporaneous donor returns are used to estimate IBKR's counterfactual during the event.
2. **Causality is not identified.** The residual cannot isolate passive demand from active trading, company-specific information or model error.
3. **Statistical power is limited.** The final holdout contains 63 sessions, and the two-session event uncertainty is supported by only 62 overlapping historical residual blocks.
4. **Daily data are coarse for index implementation.** They cannot isolate closing-auction imbalances or investor identities.
5. **Execution inputs are approximate.** Spread, commissions, market impact and borrow rely on empirical proxies or literature-based calibrations.
6. **Liquidity estimation uses a short clean post-split window.** The integrated trade-design notebook uses 46 pre-event sessions.
7. **The hedge is a return-space approximation.** PCR is linear in log returns, while an actual fixed-notional self-financing portfolio earns simple returns.

For these reasons, the final position should be interpreted as a **retrospective educational calibration**, not as evidence that the trade could have been identified or implemented in real time with the same information.

---

# Selected academic references

The notebooks draw on several strands of the literature, including:

- MacKinlay, A. C. (1997), *Event Studies in Economics and Finance*.
- Greenwood, R. and Sammon, M., *The Disappearing Index Effect*.
- Lettau, M. and Pelger, M., work on PCA/factor estimation in asset pricing.
- Corwin, S. A. and Schultz, P. (2012), *A Simple Way to Estimate Bid-Ask Spreads from Daily High and Low Prices*.
- Market-impact literature on square-root impact, including Moro et al. and Zarinelli et al.

The notebooks contain the fuller methodological discussion and source links used in the analysis.

---

# Disclaimer

This repository is an educational quantitative-finance research project.

It is **not investment advice, a live signal, or an ex-ante trading recommendation**.
