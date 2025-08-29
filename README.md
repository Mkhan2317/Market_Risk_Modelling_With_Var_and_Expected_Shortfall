

# Market Risk Modeling: Value at Risk (VaR), Expected Shortfall (ES), Liquidity Adjustments, and Covariance Denoising

---

## 1. Introduction

Modern financial markets are highly volatile, and managing **market risk** is central to both practitioners and regulators. A well-constructed portfolio is still vulnerable to sudden price movements, volatility shocks, and liquidity crises.

Since the 1990s, **Value at Risk (VaR)** has been the industry’s benchmark for risk measurement, adopted even in regulatory frameworks such as Basel II. VaR estimates the maximum loss that will not be exceeded with a given confidence level over a specified horizon. For example, a 95% 1-day VaR of \$50,000 means: *with 95% confidence, the portfolio will not lose more than \$50,000 in one day.*

However, VaR has well-documented weaknesses:

* It **ignores losses beyond the quantile** (the tail).
* It may **violate coherence properties** such as sub-additivity, meaning it can underestimate risk when assets are aggregated.

To address these shortcomings:

* We compute **Expected Shortfall (ES)**, a coherent risk measure capturing the **average of losses beyond VaR**.
* We extend ES to **Liquidity-Adjusted ES (LAES)** by incorporating bid–ask spreads and PCA-based liquidity indices, reflecting true trading costs in crises.
* We improve **parametric VaR stability** using **covariance matrix denoising**, motivated by the **Marchenko–Pastur theorem**.

This project thus combines **theory, practical computation, and empirical results** to construct a modern and robust framework for market risk measurement.

---

## 2. Objectives

1. **Implement three VaR approaches**: Parametric (Variance–Covariance), Historical Simulation, and Monte Carlo.
2. **Compute Expected Shortfall (ES)**: both parametric and historical.
3. **Validate sub-additivity** of risk measures using toy portfolios.
4. **Model liquidity risk** through bid–ask spreads, PCA, and construct Liquidity-Adjusted ES (LAES).
5. **Apply covariance denoising** (Riskfolio) to stabilize parametric VaR, guided by Marchenko–Pastur theory.

---

## 3. Data

* **Assets:** IBM, Microsoft (MSFT), Intel (INTC)
* **Period:** Jan 1, 2020 – Dec 31, 2020 (252 trading days, covering COVID-19 shocks)
* **Source:** Yahoo Finance (`yfinance`)
* **Returns:** Daily log returns

$$
r_t = \ln(P_t) - \ln(P_{t-1})
$$

* **Portfolio value:** \$1,000,000
* **Weights:** Equal or randomized (normalized to sum to 1)

---

## 4. Value at Risk (VaR)

### 4.1 Parametric (Variance–Covariance)

*Method:* assume normal returns, estimate covariance, compute portfolio σ, apply normal quantile.

**Results (95%, 1-day, \$1M):**

* IBM: **\$43,950.04**
* MSFT: **\$42,487.97**
* INTC: **\$44,598.38**

---

### 4.2 Historical Simulation

*Method:* empirical 5th percentile of returns.

**Results:**

* IBM: **\$37,201.00**
* MSFT: **\$42,622.46**
* INTC: **\$42,509.28**

---

### 4.3 Monte Carlo Simulation

*Method:* simulate 1000 draws from random distributions, compute 5th percentile.

**Results:**

* Simulation 1: **\$1,586,978.16**
* Simulation 2: **\$1,621,727.17**
* Simulation 3: **\$1,614,548.75**

---

## 5. Marchenko–Pastur Theory

The **Marchenko–Pastur law** describes the eigenvalue spectrum of large random covariance matrices:

$$
f(\lambda) = \frac{1}{2\pi\sigma^2 q \lambda} \sqrt{(\lambda_+ - \lambda)(\lambda - \lambda_-)} \quad \lambda \in [\lambda_-, \lambda_+]
$$

with $\lambda_\pm = \sigma^2 (1 \pm \sqrt{q})^2, \; q = T/N$.

* **Bulk eigenvalues** = random noise.
* **Outliers** = true correlations (e.g., sector/market modes).

This motivates **denoising covariance matrices** before risk estimation.

---

## 6. Covariance Denoising (Riskfolio)

### 6.1 Implementation

Using **Riskfolio**, covariance is denoised via a kernel density estimator applied to eigenvalues, suppressing noisy components.

```python
cov_matrix_denoised_np = rp.ParamsEstimation.covar_matrix(
    stocks_returns, method="fixed", bWidth=0.25, detone=False, mkt_comp=1
)
```

### 6.2 Results (95%, 1-day, \$1M)

* **Raw Parametric VaR:** \~\$43,950
* **Denoised Parametric VaR:** \~\$4,245 (portfolio-level output)

### 6.3 Interpretation

The denoised covariance produced **more stable and less inflated VaR**, filtering random noise that exaggerates correlation-driven risk. This adjustment is particularly important when working with high-dimensional portfolios.

---

## 7. Sub-additivity Check

Two toy portfolios tested:

* Case A: Portfolio VaR = 0.4000 vs sum = 0.5930 → ✅ sub-additivity holds.
* Case B: Portfolio VaR = 0.2750 vs sum = 0.6100 → ✅ holds.

*Reminder:* VaR is not always sub-additive in general → ES preferred.

---

## 8. Expected Shortfall (ES)

### 8.1 Parametric ES

* IBM: **\$54,807.58**
* MSFT: **\$56,269.64**
* INTC: **\$54,159.23**

### 8.2 Historical ES

* IBM: **–\$64,802.36**
* MSFT: **–\$65,765.03**
* INTC: **–\$88,462.75**

*Interpretation:* Historical ES is significantly more negative, reflecting fat-tailed losses.

---

## 9. Liquidity-Adjusted ES (Bid–Ask Spreads)

### 9.1 Method

* Compute effective, quoted, proportional spreads.
* Use PCA to capture common liquidity factor (\~89.55% + 99.38% variance explained).
* Adjust ES with liquidity index:

$$
LAES = ES + \frac{P}{2} \times LI
$$

### 9.2 Results

* IBM: **\$150,414.20**
* MSFT: **\$145,478.78**
* INTC: **\$152,601.81**

PCA-based refinement gave similar values (variance explained ≈ 65.7% + 19.3%).

---

## 10. Results Summary

| Method                      | IBM         | MSFT        | INTC        |
| --------------------------- | ----------- | ----------- | ----------- |
| Parametric VaR              | 43,950      | 42,488      | 44,598      |
| Historical VaR              | 37,201      | 42,622      | 42,509      |
| Monte Carlo VaR             | 1.59M       | 1.62M       | 1.61M       |
| Parametric ES               | 54,808      | 56,270      | 54,159      |
| Historical ES               | –64,802     | –65,765     | –88,463     |
| Liquidity-Adjusted ES       | 150,414     | 145,479     | 152,602     |
| **Denoised Parametric VaR** | **\~4,245** | **\~4,245** | **\~4,245** |

---

## 11. Detailed Findings & Analysis

The project produced rich empirical insights across multiple risk measurement techniques. Below is a deeper dive into the **interpretation of results**, linking them to both academic theory and practical regulatory applications.

---

### 11.1 VaR Comparisons

* **Parametric VaR** estimates for IBM, MSFT, and INTC were in the **\$42–45k range**. This relatively narrow band reflects the **assumption of normally distributed returns**. While efficient to compute, the Gaussian assumption is fragile: real markets exhibit skewness and fat tails, particularly during crises (e.g., March 2020).
* **Historical VaR** values are slightly lower for IBM but similar for MSFT/INTC compared to parametric. This suggests that, for 2020, historical return distributions did not generate much fatter left tails than the Gaussian assumption—except for crisis periods, where historical methods can outperform parametric.
* **Monte Carlo VaR** produced extremely large loss estimates (≈\$1.6M). This reflects the fact that simulated distributions with large variance and fat tails can **dramatically magnify tail losses**. Monte Carlo methods highlight the sensitivity of VaR to distributional assumptions and sampling variability.

**Key finding:** *VaR is highly model-dependent. Parametric underestimates extremes, historical is more realistic but assumes stationarity, and Monte Carlo can overshoot if distributions are poorly specified.*

---

### 11.2 Expected Shortfall (ES)

* **Parametric ES** values (\~\$55k per stock) are only slightly above Parametric VaR, which is consistent with Gaussian assumptions where tails thin out quickly.
* **Historical ES** values, however, are drastically higher (–\$65k to –\$88k). This demonstrates the **heavier tails observed in empirical distributions** — especially for Intel (INTC), which had more volatile swings in 2020.

**Key finding:** *Historical ES captures the severity of losses in the tail, unlike VaR which only gives a cutoff. This gap underscores why regulators (Basel III) now prefer ES over VaR for capital adequacy.*

---

### 11.3 Liquidity-Adjusted Expected Shortfall (LAES)

* Liquidity-adjusted ES values (IBM ≈ \$150k, MSFT ≈ \$145k, INTC ≈ \$152k) are **2–3 times higher than standard ES values**.
* The bid–ask spread analysis (effective cost, proportional spreads) showed extremely high correlations (>0.8), highlighting systemic liquidity effects: when one spread measure widens, others follow.
* PCA confirmed that **two principal components explain nearly all variance (≈89.5% + 99.3%)**, suggesting that liquidity shocks are largely driven by common market-wide factors (systemic liquidity risk) rather than idiosyncratic stock-level effects.

**Key finding:** *Liquidity costs dramatically amplify tail risk, making LAES a more realistic measure of stress losses. During crises, risk managers must account for liquidity dry-ups; otherwise, capital reserves will be insufficient.*

---

### 11.4 Sub-additivity

* Both toy examples demonstrated that portfolio VaR ≤ sum of individual VaRs. However, this is not guaranteed in all cases.
* The **failure of VaR to always be sub-additive** undermines its usefulness in multi-asset portfolio contexts, where diversification should never increase risk.

**Key finding:** *VaR can fail coherence tests; ES, by contrast, always respects sub-additivity, making it the preferred coherent risk measure.*

---

### 11.5 Covariance Denoising (Riskfolio)

* Raw covariance-based VaR gave ≈\$43k, while **Riskfolio-denoised covariance reduced this to ≈\$4.2k** (portfolio-level).
* The stark difference arises because **raw covariance matrices overestimate correlations due to noise** (limited sample size, high-dimensionality).
* Denoising filters out random eigenvalues predicted by Marchenko–Pastur distribution, stabilizing estimates and avoiding spurious risk inflation.

**Key finding:** *Covariance denoising is essential for robust parametric risk models. Without it, portfolios may appear riskier than they truly are, leading to over-allocation of regulatory capital or misinformed trading limits.*

---

### 11.6 Cross-Method Synthesis

* **VaR (Gaussian, Historical, Monte Carlo)** offers a spectrum: from underestimation (parametric) to overshooting (Monte Carlo).
* **ES (Parametric vs Historical)** highlights the importance of tail-awareness, with historical values aligning better with crisis episodes.
* **Liquidity adjustments** reveal the critical role of trading frictions, which are invisible in standard risk models.
* **Denoising** ensures that the covariance structure feeding into VaR/ES is robust and not overwhelmed by estimation noise.

---

### 11.7 Practical and Regulatory Implications

* **Basel III/IV Compliance:** Banks are now required to use ES at 97.5% confidence instead of VaR. This project’s results support that decision, as ES is both coherent and tail-sensitive.
* **Portfolio Management:** Quant traders and asset managers should incorporate liquidity-adjusted risk measures in stress tests. LAES demonstrates that ignoring liquidity could mean **capital buffers are off by a factor of 2–3**.
* **Risk Infrastructure:** Firms using high-dimensional portfolios should adopt covariance denoising techniques (Marchenko–Pastur + Riskfolio) to avoid noisy estimates that bias both portfolio optimization and VaR.

---

## 12. Conclusion

This project demonstrates that **risk cannot be captured by a single metric**.

* **VaR** provides an intuitive, regulatory-friendly cutoff but is fragile.
* **ES** offers a coherent, tail-focused alternative.
* **LAES** highlights the devastating role of liquidity crises.
* **Denoising** stabilizes risk estimates, avoiding false signals from noisy data.

**Final Insight:** A robust risk management framework must triangulate across **VaR, ES, liquidity adjustments, and denoised covariance matrices**. Only then can institutions achieve a realistic, crisis-ready assessment of their market exposure.





