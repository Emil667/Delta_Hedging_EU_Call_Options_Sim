# Discrete-Time Delta Hedging under Black–Scholes (Monte Carlo)

## 1) Purpose
This script studies **discrete-time delta hedging** of a **European call option** in the standard **Black–Scholes framework**.  
It simulates many price paths for the underlying under **geometric Brownian motion (GBM)**, applies a self-financing delta hedge with different rebalancing frequencies, and measures the **final hedging error**:

\[
\text{Hedging error} = V_T - (S_T - K)^+
\]

where \(V_T\) is the final value of the hedging portfolio and \((S_T - K)^+\) is the option payoff.

The key output is how the **distribution** and **variance** of the hedging error change when hedging is done:
- daily,
- weekly,
- monthly.

---

## 2) Model assumptions (why they are used)
The script uses the classical Black–Scholes assumptions because they provide a clean benchmark in which:
- the option has **closed-form price and delta**, and
- the only source of hedging error (in this experiment) is **discrete rebalancing** rather than model misspecification.

The specific assumptions are:

1. **Underlying dynamics (GBM):**
   \[
   dS_t = r S_t\,dt + \sigma S_t\,dW_t
   \]
   This implies log-returns are normally distributed with constant volatility \(\sigma\).  
   GBM is used here because it is the baseline model behind Black–Scholes and makes the delta-hedging strategy well-defined.

2. **Risk-free rate \(r\) is constant and borrowing/lending occurs at \(r\):**  
   This allows the cash account in the hedge portfolio to grow deterministically at \(\exp(r\,dt)\).  
   It is a standard simplifying assumption to isolate hedging error due to rebalancing frequency.

3. **Frictionless trading (no transaction costs, no bid-ask, unlimited liquidity):**  
   This ensures the hedge is only impacted by discretization.  
   In real markets, transaction costs create an additional trade-off: more frequent hedging reduces variance but increases costs.

4. **Continuous-time theory vs discrete implementation:**  
   Under Black–Scholes, continuous delta hedging replicates the payoff exactly.  
   In practice hedging is discrete, so the script evaluates the residual error introduced by finite rebalancing intervals.

---

## 3) Configuration (parameters and interpretation)
The “Configuration” block defines the experiment. Each parameter has a direct modeling role:

- **`S0` (initial price):** Starting underlying price at \(t=0\).  
  Used to initialize all simulated paths and to price the option at inception.

- **`K` (strike):** Strike price of the European call.  
  Determines payoff \((S_T - K)^+\).

- **`T` (maturity):** Time to maturity in years.  
  Together with `n_steps`, it defines the time increment \(dt = T/n\).

- **`r` (risk-free rate):** Constant annualized continuously-compounded rate.  
  Used both in the risk-neutral drift of GBM and in the cash account growth.

- **`sigma` (volatility):** Constant annualized volatility.  
  Controls the diffusion magnitude and is the key driver of hedging difficulty.

Simulation controls:

- **`n_steps`:** Number of time steps in \([0, T]\).  
  With `T=1` and `n_steps=252`, the grid matches a common “trading days per year” convention.

- **`n_paths`:** Number of Monte Carlo paths.  
  Larger values reduce sampling noise in the estimated error distribution.

- **`seed`:** Random seed for reproducibility.  
  Ensures results can be replicated exactly.

Hedging frequencies:

- **`freq = {"daily": 1, "weekly": 5, "monthly": 21}`**  
  Interprets hedging as rebalancing every 1, 5, or 21 time steps.  
  These values are consistent with the 252-step grid (≈ 1 year of trading days).

---

## 4) Step-by-step what the code does

### Step 1 — Normal CDF
`norm_cdf(x)` wraps SciPy’s `norm.cdf`, which is used in the Black–Scholes formulas.

### Step 2 — Black–Scholes pricing and delta
The helpers implement:
- \(d_1\)
- call price \(C(S,t)\)
- call delta \(\Delta(S,t)\)

Delta is:
\[
\Delta = N(d_1)
\]
Price is:
\[
C = S N(d_1) - K e^{-r\tau} N(d_2)
\]
where \(\tau\) is time-to-maturity.

These closed-form expressions make the hedge rule explicit and fast to compute.

### Step 3 — Simulate GBM paths under the risk-neutral measure
The script simulates log prices using:
\[
\log S_{t+\Delta t} = \log S_t + \left(r - \tfrac{1}{2}\sigma^2\right)\Delta t + \sigma \sqrt{\Delta t}\,Z
\]
with \(Z \sim \mathcal{N}(0,1)\).

This produces an array `S` of shape `(n_paths, n_steps + 1)` containing all simulated paths.

### Step 4 — Run the discrete delta hedge on each path
`hedge_error(S_paths, rebalance_every)` implements a standard self-financing hedge:

1. **Initial setup (t=0):**
   - compute initial call price \(C_0\) and delta \(\Delta_0\)
   - hold `shares = Δ0` of the underlying
   - allocate remaining value to cash:
     \[
     \text{cash}_0 = C_0 - \Delta_0 S_0
     \]

2. **Cash account accrues at the risk-free rate:**
   Each step:
   \[
   \text{cash} \leftarrow \text{cash}\cdot e^{r\,dt}
   \]

3. **Rebalance on schedule:**
   At times matching the chosen frequency:
   - compute new delta \(\Delta_t\)
   - trade \(\Delta_t - \Delta_{t^-}\) shares
   - update cash by the cost of the trade:
     \[
     \text{cash} \leftarrow \text{cash} - (\Delta_t - \Delta_{t^-})S_t
     \]

4. **Final portfolio value:**
   \[
   V_T = \Delta_T S_T + \text{cash}_T
   \]

5. **Compute hedging error:**
   \[
   V_T - (S_T - K)^+
   \]

The function returns a vector of hedging errors (one per path).

### Step 5 — Compare frequencies
The script computes hedging errors for daily, weekly, and monthly hedging, then prints:
- mean error,
- standard deviation,
- variance.

Interpretation:
- **Mean error** close to zero suggests no systematic bias under the model assumptions.
- **Variance / standard deviation** captures hedging risk induced by discrete rebalancing.

---

## 5) Outputs and interpretation

### Printed summary
For each frequency (daily/weekly/monthly), the script reports mean, std, and var of hedging errors across paths.

Expected pattern:
- **more frequent hedging ⇒ lower variance**  
  because the strategy tracks the continuous-time replicating portfolio more closely.

### Plot 1 — Variance vs hedging frequency
A bar chart showing how hedging error variance changes across frequencies.

### Plot 2 — Hedging error distributions
Overlaid histograms for daily/weekly/monthly hedging errors.
This shows:
- dispersion (risk),
- symmetry/asymmetry,
- tail behavior (large hedging losses/gains).

---

## 6) Notes / extensions (optional)
- Adding transaction costs would create an explicit frequency trade-off (risk reduction vs cost).
- Introducing model mismatch (e.g., jumps or stochastic volatility) would typically increase hedging error and may shift the mean away from zero.
- Using different `n_steps` changes the discretization grid and can be used to study numerical convergence.

---

## 7) Requirements
- Python 3.x
- NumPy
- Matplotlib
- SciPy (for `scipy.stats.norm`)

Run the script in a Python environment that supports plotting (Jupyter, Colab, or local Python).
