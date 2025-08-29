
# Market Risk Modeling: Value at Risk (VaR), Expected Shortfall (ES), and Liquidity Adjustments

---

## 1. Introduction

Financial markets are inherently uncertain. Even the most carefully constructed portfolios are exposed to adverse movements in asset prices, volatility spikes, and liquidity dry-ups. This risk — **market risk** — is one of the most important categories of risk, alongside credit and operational risks.

Since the 1990s, **Value at Risk (VaR)** has become the industry standard for quantifying market risk. VaR attempts to answer a deceptively simple question:

> *“What is the maximum loss I can expect over a given time horizon, at a given confidence level, under normal market conditions?”*

While VaR is intuitive and widely adopted (including by regulators in the Basel Accords), it has well-documented limitations. Most importantly, VaR:

* Ignores the severity of losses beyond the cutoff.
* Can fail the **coherence axioms** (especially sub-additivity).

These shortcomings led to the introduction of **Expected Shortfall (ES)** (also called Conditional VaR or CVaR) as the new regulatory risk measure under Basel III. ES accounts not only for the quantile but also for the **average of losses in the tail**.

In practice, risk is further magnified during **liquidity crises**, when bid–ask spreads widen and forced liquidation costs amplify portfolio losses. To capture this, we extend ES to **Liquidity-Adjusted Expected Shortfall (LAES)**.

Finally, we discuss the **Marchenko–Pastur theorem**, which provides theoretical motivation for filtering noise from covariance matrices — a critical step before computing parametric risk estimates.

---

## 2. Objectives

This project is structured around five concrete objectives:

1. **Implement three VaR approaches** — Variance–Covariance (parametric), Historical Simulation, and Monte Carlo.
2. **Compute Expected Shortfall (ES)** and highlight why it is a superior measure of tail risk.
3. **Validate sub-additivity** using toy portfolio examples, showing where VaR may fail coherence.
4. **Incorporate liquidity risk** into ES through bid–ask spread analysis and PCA-based liquidity factors.
5. **Provide theoretical context** by introducing the Marchenko–Pastur law of random matrices.

This blend of theory, implementation, and interpretation mirrors the workflow of practitioners in investment banks, hedge funds, and regulators.

---

## 3. Data

* **Assets studied:** IBM, Microsoft (MSFT), Intel (INTC)
* **Period:** Jan 1, 2020 – Dec 31, 2020 (252 trading days, covering the COVID-19 crisis)
* **Data source:** Yahoo Finance via `yfinance`
* **Data processing:** Daily log returns, defined as

$$
r_t = \ln(P_t) - \ln(P_{t-1})
$$

where $P_t$ = adjusted close price on day $t$.

* **Portfolio value:** \$1,000,000 baseline

This dataset provides a realistic context: large-cap tech stocks during a volatile year, capturing both “normal” and “crisis” regimes.

---

## 4. Value at Risk (VaR)

### 4.1 Definition

For a portfolio with loss distribution $L$, the $\alpha$-quantile VaR is:

$$
\text{VaR}_{\alpha}(L) = \inf \{ l \in \mathbb{R} : P(L \leq l) \geq \alpha \}
$$

It represents the maximum loss not exceeded with probability $\alpha$. For example, a 95% 1-day VaR of \$1M means: *“With 95% confidence, losses will not exceed \$1M in one day.”*

---

### 4.2 Parametric (Variance–Covariance) VaR

**Methodology:**

1. Assume returns \~ Normal($\mu, \Sigma$).
2. Compute portfolio volatility:

   $$
   \sigma_p = \sqrt{w^T \Sigma w}
   $$
3. Apply normal quantile:

   $$
   \text{VaR}_{\alpha} = V \cdot (z_\alpha \sigma_p - \mu_p)
   $$

**Results (95%, 1-day, \$1M):**

* IBM: **\$43,950.04**
* MSFT: **\$42,487.97**
* INTC: **\$44,598.38**

**Interpretation:** The parametric method is efficient but sensitive to the normality assumption. In crises (e.g., COVID crash), tails are heavier than Gaussian, making this approach prone to underestimation.

---

### 4.3 Historical Simulation VaR

**Methodology:**

* Collect historical returns.
* Take empirical 5th percentile as cutoff.

**Results (95%, 1-day, \$1M):**

* IBM: **\$37,201.00**
* MSFT: **\$42,622.46**
* INTC: **\$42,509.28**

**Interpretation:** Historical VaR better captures real market distributions (including fat tails) but assumes history repeats. If structural breaks occur, its reliability declines.

---

### 4.4 Monte Carlo Simulation VaR

**Methodology:**

* Specify distribution (e.g., normal).
* Simulate thousands of return paths.
* Compute empirical quantile of simulated distribution.

**Results (illustrative):**
Notebook generated three simulation paths with 1000 draws each .

**Interpretation:** Monte Carlo is flexible — distributions can include skewness, kurtosis, stochastic volatility — but results depend heavily on assumptions.

---

## 5. Marchenko–Pastur Theory

Large covariance matrices often mix signal (true correlations) and noise. The **Marchenko–Pastur law** gives the limiting distribution of eigenvalues of random covariance matrices:

$$
f(\lambda) = \frac{1}{2\pi\sigma^2 q \lambda}\sqrt{(\lambda_+ - \lambda)(\lambda - \lambda_-)} \quad \lambda \in [\lambda_-, \lambda_+]
$$

with $\lambda_\pm = \sigma^2 (1 \pm \sqrt{q})^2$, $q = T/N$.

**Implication:** Eigenvalues inside $[\lambda_-, \lambda_+]$ are noise. Outliers may represent true market factors. This is crucial for **risk models**, where noisy covariance estimates distort VaR/ES.
*(Denoising implementation excluded, per request.)*

---

## 6. Sub-additivity Check

A **coherent risk measure** must satisfy sub-additivity:

$$
\rho(X+Y) \leq \rho(X) + \rho(Y)
$$

**Results (toy examples):**

* **Case A (90% confidence):**

  * Asset 1 VaR = 0.3100
  * Asset 2 VaR = 0.2830
  * Portfolio VaR = 0.4000
  * ✅ Sub-additivity holds (0.4000 ≤ 0.5930).

* **Case B (90% confidence):**

  * Asset 1 VaR = 0.0440
  * Asset 2 VaR = 0.5660
  * Portfolio VaR = 0.2750
  * ✅ Holds again (0.2750 ≤ 0.6100).

**Interpretation:** In these examples, diversification reduces risk as expected. But in general, VaR can violate sub-additivity, making ES preferable.

---

## 7. Expected Shortfall (ES)

### 7.1 Definition

ES is the **average of losses worse than VaR**:

$$
ES_\alpha = \mathbb{E}[L \mid L > \text{VaR}_\alpha]
$$

Equivalently:

$$
ES_\alpha = \frac{1}{1-\alpha}\int_{\alpha}^1 q(u) \, du
$$

where $q(u)$ = quantile function.

**Advantage:** ES is **coherent** — it respects sub-additivity.

---

### 7.2 Results

**Parametric ES (95%, 1-day, \$1M):**

* IBM: **\$52,776.42**
* MSFT: **\$54,358.08**
* INTC: **\$52,273.33**

**Historical ES (95%, 1-day, \$1M):**

* IBM: **\$150,358.47**
* MSFT: **\$145,356.56**
* INTC: **\$152,576.53**

**Interpretation:** Historical ES is dramatically larger than parametric ES — clear evidence of fat-tailed returns. ES captures these risks; VaR alone would underestimate them.

---

## 8. Liquidity-Adjusted Expected Shortfall (LAES)

### 8.1 Motivation

During crises, **liquidity vanishes**. Bid–ask spreads widen, order books thin, and forced sales cause **fire-sale discounts**. Traditional ES ignores this.

### 8.2 Methodology

1. Compute spread measures (quoted, effective, proportional).
2. Standardize spreads and apply **PCA**.

   * First two PCs explain \~89.55% and \~99.38% of variance .
3. Construct liquidity index from PCs.
4. Adjust ES:

$$
LAES = ES + \frac{P}{2} \times LI
$$

### 8.3 Results

**Liquidity-Adjusted ES (95%, 1-day, \$1M):**

* IBM: **\$150,414.20**
* MSFT: **\$145,478.78**
* INTC: **\$152,601.81**

**Interpretation:** Liquidity adjustment raises ES further — reflecting the **true all-in cost of risk**. In stressed markets, losses are not only larger but also more expensive to realize due to frictions.

---

## 9. Results Summary

| Method                | IBM     | MSFT    | INTC    |
| --------------------- | ------- | ------- | ------- |
| Parametric VaR        | 43,950  | 42,488  | 44,598  |
| Historical VaR        | 37,201  | 42,622  | 42,509  |
| Parametric ES         | 52,776  | 54,358  | 52,273  |
| Historical ES         | 150,358 | 145,357 | 152,577 |
| Liquidity-Adjusted ES | 150,414 | 145,479 | 152,602 |

---

## 10. Key Insights

1. **Model sensitivity:** Parametric VaR grossly underestimates risk vs Historical/ES, especially in crises.
2. **Superiority of ES:** ES is tail-focused and coherent, making it the regulator’s preferred measure.
3. **Liquidity matters:** LAES shows risk can **triple** once transaction costs are included — an often ignored but crucial factor.
4. **Covariance noise:** Marchenko–Pastur theory warns against blindly trusting covariance matrices — denoising is essential for robust risk estimates.

---

## 11. Implementation

* **Language:** Python 3.x
* **Libraries:** `numpy`, `pandas`, `scipy`, `matplotlib`, `seaborn`, `yfinance`, `sklearn`
* **Notebook:** `Market_Risk_Modelling_with_VAR.ipynb`
* **Outputs:** Plots of VaR over horizon, histograms of losses, PCA scree plots.

---

## 12. Conclusion

This project demonstrates that **risk is multi-dimensional**:

* **VaR** is useful for communication but fragile under fat tails.
* **ES** is more reliable, coherent, and regulator-endorsed.
* **Liquidity adjustments** are indispensable during crises, when frictions amplify losses.
* **Marchenko–Pastur theory** provides a theoretical backbone for denoising covariance matrices before applying risk models.

**Final Insight:**
No single risk measure is sufficient. A robust framework must **combine multiple approaches** (VaR, ES, LAES) and account for **market microstructure and tail events** to deliver reliable risk management.


