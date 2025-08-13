# Cycle-Aware Probabilistic Stock Demo

This project demonstrates a **cycle-centric** workflow for probabilistic stock forecasting:
we first use **FFT** to detect the dominant cycle and choose a **stable observation window**,
then estimate **probabilities of up/down X%** and **probability to reach the cycle high**
with **four distribution engines** (Price-Normal, Log-Normal, Student’s *t*, Historical Simulation).
Finally, a **rolling backtest** automatically selects the best-performing engine.

> This is a **public demo repository** — the full code and pipelines remain **private**.  
> For collaboration or access requests, please contact me.

---

## ✨ Features

- **FFT cycle detection**: de-mean → FFT → pick peak frequency → convert to **period (days)**
- **Best observation window X**: scan windows and pick the one whose dominant period is most **stable** (closest to an integer)
- **Return-driven period**: inside a candidate range, choose the period that maximizes \((\max - \min)/\min\)
- **Phase progress**: anchor at the most recent low; compute in-cycle time progress and phase angle
- **Four probability engines**:  
  0) **Price-Normal**, 1) **Log-Normal**, 2) **Student’s t** (fat tails), 3) **Historical Simulation** (non-parametric)
- **Automatic model selection**: multi-threshold **Brier Score** with **rolling backtest**
- **Indicator vote (demo)**: MA / RSI / Bollinger / OBV for contextual tags
- **Outputs**: trendlines, FFT spectrum, probability curves & tables (see `output/`)

---

## 🧱 Pipeline

1. Download market data (e.g., Yahoo Finance)  
2. Preprocess (de-meaned series, indicators, **daily log-returns**)  
3. **FFT** → dominant period & **best observation window X**  
4. **Return-driven** period selection (maximize amplitude return in candidate set)  
5. **Phase progress**: locate today within the cycle  
6. **Four probability engines** for the terminal price \(S_T\)  
7. **Rolling backtest** (multi-threshold events, Brier Score) → **select best engine**  
8. Produce **probabilities for up/down X%** and **probability to reach the high**, with composite plots

**Text-only flow (pasteable):**

```

                      [Tickers / Data]
                             │
                             ▼
                 [Preprocess & Indicators]
                             │
                             ▼
                 [FFT (Frequency → Period)]
                             │
                             ▼
                [Best Observation Window X]
                             │
                             ▼
                  [Return-Driven Period]
                             │
                             ▼
                     [Phase Progress]
                             │
                             ▼
 [Probability Engines: Price-Normal | Log-Normal | Student t | Historical]
                             │
                             ▼
           [Rolling Backtest → Find the Best Model]
                             │
                             ▼
             [Probabilities: Up/Down x% & Reach High]

```


---

## 📊 Demo Outputs (examples)

- FFT spectrum & dominant period: `output/fft_spectrum.png`  
  ![FFT](output/fft.png)

- Up/Down probability printout: `output/probabilities_example.png`  
  ![Probabilities](output/probabilities_example.png)

- Cycle metrics & phase progress: `output/phase_progress.png`  
  ![Phase](output/phase_progress.png)

- (Optional) Trendlines with signals: `output/trendlines_signals.png`  
- (Optional) Model selection (Brier comparison): `output/model_selection_brier.png`  

> Replace the filenames above with the actual artifacts inside your `output/` folder.

---

## 🧮 Math Notes (Core Formulas)

### 1) FFT Dominant Period (Frequency → Period)
Apply DFT to the **de-meaned** price series \(x_t - \bar{x}\), keep **positive frequencies**, and pick the peak frequency \(\hat{f}\).  
The dominant **period** (trading days per cycle):
\[
\hat{P} = \frac{1}{\hat{f}}.
\]

### 2) Best Observation Window \(X\)
Scan \(X \in [30, 200]\) (configurable step). For each window, run FFT to obtain \(\hat{P}_X\).  
Define **stability** by proximity of \(\hat{P}_X\) to an integer:
\[
\text{frac} = \hat{P}_X - \lfloor \hat{P}_X \rfloor,\quad
\text{stability}(X)=
\begin{cases}
+\infty & \text{if } \text{frac}=0,\\[4pt]
\frac{1}{\min(\text{frac},1-\text{frac})} & \text{otherwise.}
\end{cases}
\]
Pick \(X\) with the largest stability.

### 3) Return-Driven Period (within candidates)
For each candidate \(P\) (e.g., 15–45 days), take the most recent \(P\) days:
\[
\text{Return}(P)=\frac{\max(S)-\min(S)}{\min(S)}\times 100\%.
\]
Choose the \(P\) that maximizes \(\text{Return}(P)\).

### 4) Phase Progress (Time within the cycle)
Using the **most recent local low** within the latest period window as a phase anchor:
\[
\text{progress}=\frac{(\text{days\_since\_last\_low})\bmod P}{P},\qquad
\theta=2\pi\cdot \text{progress},
\]
where \(\theta\) is the phase angle and \(\text{progress}\in[0,1)\) is the cycle time progress.

### 5) Annualized Volatility → \(H\)-Day Horizon
With annualized volatility \(\sigma_{\text{ann}}\) from **daily log-returns**, scale to \(H\) days:
\[
\sigma_H=\sigma_{\text{ann}}\sqrt{\frac{H}{252}},
\]
where **252** ≈ trading days per year.

### 6) Probability Engines (for terminal price \(S_T\))

**(0) Price-Normal (simplified):** assume \(S_T \sim \mathcal{N}(\mu_S,\sigma_S)\),
\[
\mu_S = S_0 (1 + \text{ER}),\qquad
\sigma_S = S_0 \cdot \sigma_{\text{ann}}\sqrt{\tfrac{H}{252}}.
\]
Then
- \(K \ge S_0\): \(\mathbb{P}(S_T\ge K)=1-\Phi\!\bigl(\tfrac{K-\mu_S}{\sigma_S}\bigr)\)
- \(K \le S_0\): \(\mathbb{P}(S_T\le K)=\Phi\!\bigl(\tfrac{K-\mu_S}{\sigma_S}\bigr)\)

**(1) Log-Normal (normal log-returns — recommended):** let \(r=\ln(S_T/S_0)\sim \mathcal{N}(\mu_r,\sigma_r)\),  
\(\sigma_r=\sigma_{\text{ann}}\sqrt{\tfrac{H}{252}},\ \mu_r\approx\ln(1+\text{ER})\). Then
\[
\mathbb{P}(S_T\ge K)=1-\Phi\!\Bigl(\frac{\ln(K/S_0)-\mu_r}{\sigma_r}\Bigr),\quad
\mathbb{P}(S_T\le K)=\Phi\!\Bigl(\frac{\ln(K/S_0)-\mu_r}{\sigma_r}\Bigr).
\]

**(2) Student’s *t* (log-returns, fat tails):** \(r=\ln(S_T/S_0)\sim t_{\nu}(\mu_r,\sigma_r)\).  
Replace \(\Phi\) by the Student-*t* CDF for better tail behavior.

**(3) Historical Simulation (non-parametric):** use past **\(H\)-day log-return** samples in a long window (e.g., 3 years) to form the empirical distribution:
\[
\widehat{\mathbb{P}}(S_T\ge K)=\frac{1}{N}\sum_{i=1}^N \mathbf{1}\!\left\{r_i \ge \ln(K/S_0)\right\}.
\]

> **ER (Expected Return)** can be set heuristically from **remaining upside × remaining time in the cycle**; log-models use \(\mu_r=\ln(1+\text{ER})\) for consistency.

### 7) Event Probabilities (Up/Down \(x\%\), Reach Cycle High)
- Up \(x\%\): set \(K=S_0(1+x)\) → compute \(\mathbb{P}(S_T\ge K)\)  
- Down \(x\%\): set \(K=S_0(1-x)\) → compute \(\mathbb{P}(S_T\le K)\)  
- Reach cycle high: set \(K=\text{cycle\_high}\)

### 8) Model Selection (Rolling Backtest + Brier Score)
For each evaluation date \(t\), with horizon \(H\) and thresholds \(x\in\{2\%,5\%,10\%\}\), compare predicted probabilities \(p_t\) vs. realized binary outcomes \(y_t\):
\[
\text{Brier}=\frac{1}{n}\sum_{i=1}^n (p_i - y_i)^2\quad \text{(averaged across multiple thresholds)}.
\]
Choose the engine with the **lowest average Brier** as the production model.

---

##  Repository Structure


```
Repo
├─ output/
│ ├─ fft_spectrum.png
│ ├─ probabilities_example.png
│ ├─ phase_progress.png
└─ README.md

```

---
Contact
For full implementation or custom adaptations:

Kai Yeh
📧 Email: KaiYeh820206@gmail.com
💻 GitHub: WayneKaiYeh
---
📄 License
https://licensebuttons.net/l/by-nc-nd/4.0/88x31.png

Allowed: Personal/educational use with attribution

Prohibited: Commercial use or redistribution

No Derivatives: Modified versions not permitted

Full license terms: Creative Commons BY-NC-ND 4.0
