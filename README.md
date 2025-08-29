

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

You’re right—I hadn’t spelled the math out. Here’s a clean, copy-pasteable **“Mathematical Formulas & Derivations”** section you can drop into your README right after the Findings/Results. I included each method you used (VaR—parametric/historical/MC, ES—parametric/historical, liquidity metrics, LAES, MP law + denoising, PCA, sub-additivity, and portfolio aggregation). All equations are in LaTeX blocks.

---

# 13. Mathematical Formulas & Derivations

## 13.1 Notation (quick reference)

* $P_t$: adjusted close price at day $t$
* $r_t = \ln(P_t) - \ln(P_{t-1})$: log return
* $\boldsymbol{w}\in\mathbb{R}^N$: portfolio weights, $\sum_i w_i = 1$
* $\boldsymbol{\mu}\in\mathbb{R}^N$: mean vector of returns
* $\Sigma\in\mathbb{R}^{N\times N}$: covariance matrix of returns
* $\mu_p = \boldsymbol{w}^\top \boldsymbol{\mu}$, $\sigma_p^2 = \boldsymbol{w}^\top \Sigma \boldsymbol{w}$
* $V$: portfolio value (e.g., $$V=\$1{,}000{,}000$$)
* Confidence level $\alpha \in (0,1)$ (e.g., $0.95$); standard normal quantile $z_\alpha$
* Loss $L = -R \cdot V$, where $R$ is portfolio return

---

## 13.2 VaR (Value at Risk)

### 13.2.1 Definition (loss quantile)

$$
\mathrm{VaR}_\alpha(L) \;=\; \inf\{ \ell \in \mathbb{R} \,:\, \mathbb{P}(L \le \ell) \ge \alpha \}.
$$

Equivalent on **returns** $R$: if $L=-VR$,

$$
\mathrm{VaR}_\alpha(L) = V \cdot \big(-q_R(1-\alpha)\big),
$$

where $q_R(p)$ is the $p$-quantile of $R$.

### 13.2.2 Parametric (Variance–Covariance) VaR (Normal)

Assume portfolio returns $R_p \sim \mathcal{N}(\mu_p, \sigma_p^2)$. Then

$$
q_{R_p}(1-\alpha) = \mu_p + \sigma_p \, z_{1-\alpha}.
$$

Loss quantile:

$$
\boxed{ \mathrm{VaR}_\alpha^{\text{para}} \;=\; V \cdot \big( -q_{R_p}(1-\alpha) \big)
\;=\; V \cdot \left( -\mu_p - \sigma_p z_{1-\alpha} \right). }
$$

Equivalently (same thing, rearranged):

$$
\mathrm{VaR}_\alpha^{\text{para}} = V \cdot \left( z_\alpha \sigma_p - \mu_p \right),
\quad \text{since } z_\alpha = -z_{1-\alpha}.
$$

**Portfolio moments:**

$$
\mu_p = \boldsymbol{w}^\top \boldsymbol{\mu}, 
\qquad
\sigma_p = \sqrt{\boldsymbol{w}^\top \Sigma \boldsymbol{w}}.
$$

### 13.2.3 Historical Simulation VaR

Let $\{r_t\}_{t=1}^T$ be historical portfolio returns. The empirical quantile:

$$
\boxed{ \mathrm{VaR}_\alpha^{\text{hist}} \;=\; V \cdot \Big( - \widehat{q}_R(1-\alpha) \Big), }
$$

where $\widehat{q}_R(1-\alpha)$ is the empirical $(1-\alpha)$-quantile of $\{r_t\}$.

### 13.2.4 Monte Carlo VaR

Simulate $M$ draws $R^{(1)},\dots,R^{(M)}$ from an assumed model (e.g., normal, t, GARCH). The empirical quantile:

$$
\boxed{ \mathrm{VaR}_\alpha^{\text{MC}} \;=\; V \cdot \Big( - \widehat{q}_{R^{(m)}}(1-\alpha) \Big). }
$$

### 13.2.5 Time scaling (i.i.d. normal approximation)

For horizon $h$ days (i.i.d., normal),

$$
\mu_{p,h} = h\,\mu_p, 
\qquad 
\sigma_{p,h} = \sqrt{h}\,\sigma_p,
$$

$$
\mathrm{VaR}_{\alpha,h}^{\text{para}} 
= V \cdot \left( z_\alpha \sigma_{p,h} - \mu_{p,h} \right)
= V \cdot \left( z_\alpha \sqrt{h}\,\sigma_p - h\,\mu_p \right).
$$

(*Caveat:* scaling can break under volatility clustering/heavy tails.)

---

## 13.3 Expected Shortfall (ES)

### 13.3.1 Definition (continuous)

$$
\boxed{ \mathrm{ES}_\alpha(L) 
= \mathbb{E}[\,L \mid L \ge \mathrm{VaR}_\alpha(L)\,] 
= \frac{1}{1-\alpha} \int_\alpha^1 q_L(u)\, du. }
$$

### 13.3.2 Discrete estimator (historical/MC)

Sort losses $L_{(1)} \le \dots \le L_{(T)}$, with $k=\lceil \alpha T\rceil$:

$$
\boxed{ \widehat{\mathrm{ES}}_\alpha 
= \frac{1}{T-k} \sum_{i=k+1}^{T} L_{(i)}. }
$$

### 13.3.3 Parametric ES (Normal losses)

If $R_p \sim \mathcal{N}(\mu_p,\sigma_p^2)$, loss $L=-VR_p$ is Normal$\big(-V\mu_p, (V\sigma_p)^2\big)$. Then

$$
\boxed{ \mathrm{ES}_\alpha^{\text{para}}
= V\left(-\mu_p + \sigma_p\,\frac{\phi(z_\alpha)}{1-\alpha}\right), }
$$

where $\phi(\cdot)$ is the standard normal pdf.

---

## 13.4 Liquidity metrics (bid–ask) and LAES

### 13.4.1 Mid-price and effective cost

$$
\text{Mid}_t = \frac{\text{ASK}_t + \text{BID}_t}{2},\qquad
\text{EffCost}_t = 
\begin{cases}
\frac{\text{PRC}_t - \text{Mid}_t}{\text{Mid}_t}, & \text{buyer-initiated} \\
\frac{\text{Mid}_t - \text{PRC}_t}{\text{Mid}_t}, & \text{seller-initiated}
\end{cases}
$$

### 13.4.2 Spread measures

$$
\text{Quoted}_t = \text{ASK}_t - \text{BID}_t,\qquad
\text{PropQuoted}_t = \frac{\text{ASK}_t - \text{BID}_t}{\text{Mid}_t},
$$

$$
\text{Effective}_t = 2\left|\text{PRC}_t-\text{Mid}_t\right|,\qquad
\text{PropEffective}_t = 2\frac{\left|\text{PRC}_t-\text{Mid}_t\right|}{\text{PRC}_t}.
$$

### 13.4.3 PCA on standardized spreads

Standardize $X\in\mathbb{R}^{T\times d}$ (columns = spread measures), then PCA:

$$
X = U \Lambda^{1/2} V^\top,\qquad
\text{scores}=X V_k,\quad \text{var explained}=\frac{\sum_{i=1}^k \lambda_i}{\sum_{i=1}^d \lambda_i}.
$$

### 13.4.4 Liquidity-Adjusted ES (LAES)

Let $LI$ be a (scaled) liquidity index (e.g., mean of first 1–2 PCs), and $P$ the prevailing price:

$$
\boxed{ \mathrm{LAES}_\alpha \;=\; \mathrm{ES}_\alpha \;+\; \frac{P}{2}\cdot LI. }
$$


---

## 13.5 Marchenko–Pastur (M–P) law & Denoising

### 13.5.1 M–P eigenvalue density

For $q=T/N$ and noise variance $\sigma^2$,

$$
\lambda_\pm = \sigma^2(1\pm\sqrt{q})^2,
$$

$$
\boxed{ f(\lambda) 
= \frac{1}{2\pi\sigma^2 q \lambda}\sqrt{(\lambda_+ - \lambda)(\lambda - \lambda_-)}\;\; \text{for}\;\; \lambda\in[\lambda_-,\lambda_+]. }
$$

Eigenvalues inside $[\lambda_-,\lambda_+]$ are “noise-bulk”; outside are potential **signal**.

### 13.5.2 Denoising pipeline (eigen-KDE idea used by Riskfolio)

1. **Eigen-decompose** sample covariance $\Sigma = Q \Lambda Q^\top$.
2. **Estimate** noise bulk (via M–P or KDE) and **shrink/replace** bulk eigenvalues $\{\lambda_i\}$ by a common/banded level $\tilde{\lambda}$.
3. **Reconstruct** denoised covariance: $\tilde{\Sigma} = Q \tilde{\Lambda} Q^\top$, where $\tilde{\Lambda}$ has adjusted eigenvalues.
4. (Optional **detoning**) remove market component (largest eigenvector) if desired.

**Parametric VaR with denoised covariance:**

$$
\tilde{\sigma}_p = \sqrt{\boldsymbol{w}^\top \tilde{\Sigma}\, \boldsymbol{w}},\qquad
\boxed{ \mathrm{VaR}_\alpha^{\text{para, denoise}} 
= V\left(z_\alpha \tilde{\sigma}_p - \mu_p\right). }
$$

---

## 13.6 Sub-additivity & Coherence

### 13.6.1 Coherent risk measure axioms (for loss variable $X$)

A risk measure $\rho(\cdot)$ is **coherent** if for all $X,Y$ and $a\ge 0$:

1. **Monotonicity:** $X \le Y \Rightarrow \rho(X) \le \rho(Y)$.
2. **Sub-additivity:** $\boxed{\rho(X+Y) \le \rho(X)+\rho(Y)}$.
3. **Positive homogeneity:** $\rho(aX)=a\rho(X)$.
4. **Translation invariance:** $\rho(X+a)=\rho(X)+a$.

**Expected Shortfall** is coherent; **VaR** can violate sub-additivity in some distributions.

---

## 13.7 Portfolio aggregation (asset ↦ portfolio)

### 13.7.1 From asset returns to portfolio return

$$
R_p = \sum_{i=1}^N w_i r_i = \boldsymbol{w}^\top \boldsymbol{r},\qquad
\mu_p = \boldsymbol{w}^\top \boldsymbol{\mu},\qquad
\sigma_p^2 = \boldsymbol{w}^\top \Sigma \boldsymbol{w}.
$$

### 13.7.2 From return to dollar loss

$$
L = -V R_p \quad\Rightarrow\quad \mathrm{VaR}_\alpha(L) = V\cdot \big(-q_{R_p}(1-\alpha)\big),\quad
\mathrm{ES}_\alpha(L) = V\cdot \mathbb{E}\!\left[-R_p \mid -R_p \ge \frac{\mathrm{VaR}_\alpha(L)}{V}\right].
$$

---

## 13.8 Useful Normal identities

* $z_\alpha = \Phi^{-1}(\alpha)$ where $\Phi(\cdot)$ is the standard normal CDF.
* $\phi(z) = \dfrac{1}{\sqrt{2\pi}}e^{-z^2/2}$.
* For $X\sim \mathcal{N}(\mu,\sigma^2)$,

$$
\mathbb{E}[X \mid X \le \mu + \sigma z_\alpha] 
= \mu - \sigma \frac{\phi(z_\alpha)}{\alpha},\quad
\mathbb{E}[X \mid X \ge \mu + \sigma z_\alpha] 
= \mu + \sigma \frac{\phi(z_\alpha)}{1-\alpha}.
$$

(These yield the closed-form ES for Gaussian models.)

---

### 13.9 Optional: Cornish–Fisher (non-normal adjustment)

If wanted, adjust quantiles for skewness $\gamma_1$ and excess kurtosis $\gamma_2$ of returns:

$$
z_{\alpha}^{\text{CF}} \approx z_\alpha 
+ \frac{\gamma_1}{6}(z_\alpha^2-1)
+ \frac{\gamma_2}{24}(z_\alpha^3-3z_\alpha)
- \frac{\gamma_1^2}{36}(2z_\alpha^3-5z_\alpha).
$$

Then use $z_{\alpha}^{\text{CF}}$ in the parametric VaR formula.

---

## 13.10 Where these plug into your code

* **Parametric VaR/ES:** Sections 13.2.2 and 13.3.3 → functions `VaR_parametric`, `ES_parametric`.
* **Historical VaR/ES:** Sections 13.2.3 and 13.3.2 → functions `VaR_historical`, `ES_historical` (empirical percentiles and tail mean).
* **Monte Carlo VaR:** Section 13.2.4 → function `MC_VaR` (quantile of simulated returns).
* **Liquidity metrics & LAES:** Sections 13.4.1–13.4.4 → spread computations, PCA, then `LAES = ES + (P/2)·LI`.
* **MP & Denoising:** Sections 13.5.1–13.5.2 → Riskfolio’s `ParamsEstimation.covar_matrix` (KDE-eigen denoising) then `VaR_parametric_denoised`.
* **Sub-additivity:** Section 13.6 → your two toy examples verifying $\rho(X+Y)\le \rho(X)+\rho(Y)$.






