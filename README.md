# Discrete-Time Delta Hedging under Black–Scholes (Monte Carlo)

## 1) Purpose
This script studies **discrete-time delta hedging** of a **European call option** in the standard **Black–Scholes framework**. It simulates many price paths for the underlying under **geometric Brownian motion (GBM)**, applies a self-financing delta hedge with different rebalancing frequencies, and measures the final **hedging error**:

$$
\text{Hedging error} = V_T - (S_T - K)^+
$$

where \(V_T\) is the terminal value of the hedging portfolio and \((S_T - K)^+\) is the option payoff.

The main objective is to see how the **distribution** and **variance** of the hedging error change when hedging is done:
- daily,
- weekly,
- monthly.

---

## 2) Model assumptions (why they are used)
The code uses the Black–Scholes assumptions because they give a clean benchmark where the option has a closed-form **price** and **delta**, and where (in theory) continuous-time delta hedging replicates the payoff exactly. In this setting, any residual hedging error in the script is driven primarily by **discrete rebalancing** rather than model ambiguity.

Key assumptions:

1. **Underlying dynamics (GBM under the risk-neutral measure)**  
   The underlying price follows:
$$
dS_t = r S_t\,dt + \sigma S_t\,dW_t
$$
   GBM is used because it is the standard Black–Scholes model: it implies log-returns are conditionally normal and volatility \(\sigma\) is constant.

2. **Constant risk-free rate and financing at \(r\)**  
   The cash account accrues deterministically at the risk-free rate. This makes the hedge bookkeeping transparent and isolates the effect of rebalancing frequency.

3. **Frictionless markets (no transaction costs, no bid–ask, infinite liquidity)**  
   These assumptions remove trading frictions. The goal is not realism here; it is to quantify the mechanical discretization error from rebalancing less frequently.

4. **Continuous-time replication vs discrete-time implementation**  
   Under Black–Scholes, continuous delta hedging replicates the option payoff (zero hedging error in the ideal limit). In discrete time, replication is imperfect, which is exactly what the script measures.

---

## 3) Configuration (parameters and interpretation)
The “Configuration” block defines a baseline option and simulation environment:

- `S0` — initial underlying price \(S_0\)  
- `K` — strike price \(K\)  
- `T` — maturity in years \(T\)  
- `r` — constant risk-free rate \(r\) (continuously compounded)  
- `sigma` — constant volatility \(\sigma\)

Simulation controls:

- `n_steps` — number of time steps in \([0, T]\)  
  The time increment is:
  $$
  \Delta t = \frac{T}{n_{\text{steps}}}
  $$
  With `T = 1` and `n_steps = 252`, the grid matches the common “trading days per year” convention.

- `n_paths` — number of Monte Carlo paths  
  Larger values reduce Monte Carlo sampling noise in the estimated error distribution.

- `seed` — random seed  
  Ensures reproducibility.

Hedging frequencies:

- `freq = {"daily": 1, "weekly": 5, "monthly": 21}`  
  Interprets rebalancing every 1, 5, or 21 time steps, consistent with a 252-step year.

---

## 4) Step-by-step: what the code does

### Step 1 — Normal CDF
`norm_cdf(x)` uses SciPy’s standard normal CDF \(N(\cdot)\), required for Black–Scholes pricing and delta.

### Step 2 — Black–Scholes price and delta
The script defines:

- \(d_1\):
$$
d_1 = \frac{\ln(S/K) + (r + \tfrac{1}{2}\sigma^2)\tau}{\sigma\sqrt{\tau}}
$$
- \(d_2 = d_1 - \sigma\sqrt{\tau}\)

For a call option, the Black–Scholes price is:
$$
C(S,\tau) = S\,N(d_1) - K e^{-r\tau} N(d_2)
$$

and the delta is:
$$
\Delta(S,\tau) = N(d_1)
$$

where \(\tau\) is time-to-maturity.

### Step 3 — Simulate GBM price paths
The code simulates log-prices using the discrete-time GBM form:
$$
\ln S_{t+\Delta t} = \ln S_t + \left(r - \tfrac{1}{2}\sigma^2\right)\Delta t + \sigma\sqrt{\Delta t}\,Z
$$
with 
$$
\(Z \sim \mathcal{N}(0,1)\).  
$$
This produces an array `S` with shape `(n_paths, n_steps + 1)`.

### Step 4 — Apply discrete-time delta hedging
`hedge_error(S_paths, rebalance_every)` runs a self-financing hedge along each simulated path:

1. **Initialize at \(t=0\)**  
   Compute the option price \(C_0\) and delta \(\Delta_0\).  
   Hold \(\Delta_0\) shares and put the remainder in cash:
$$
\text{cash}_0 = C_0 - \Delta_0 S_0
$$

2. **Accrue cash at the risk-free rate**  
   Each step:
$$
\text{cash} \leftarrow \text{cash}\cdot e^{r\Delta t}
$$

3. **Rebalance on schedule**  
   When \(t\) hits the rebalance grid:
   - compute the new delta \(\Delta_t\),
   - trade \(\Delta_t - \Delta_{t^-}\) shares,
   - update cash by the trade cost:
$$
\text{cash} \leftarrow \text{cash} - (\Delta_t - \Delta_{t^-})S_t
$$

4. **Compute terminal portfolio value and hedging error**  
   Terminal portfolio:
$$
V_T = \Delta_T S_T + \text{cash}_T
$$
   Hedging error:
$$
V_T - (S_T - K)^+
$$

The function returns a vector of hedging errors (one per path).

### Step 5 — Compare frequencies
The script runs the hedge for daily, weekly, and monthly rebalancing and prints:
- mean hedging error,
- standard deviation,
- variance.

Interpretation:
- A mean near zero indicates little systematic bias under the model.
- Lower variance indicates tighter replication (more effective hedging in discrete time).

---

## 5) Outputs
1. **Console summary** (mean, std, var of hedging error for each frequency).
2. **Bar chart**: variance of hedging error vs hedging frequency.
3. **Histogram overlay**: hedging error distributions for daily/weekly/monthly rebalancing.

---

## 6) Requirements
- Python 3.x  
- NumPy  
- Matplotlib  
- SciPy (for `scipy.stats.norm`)

Run the script in an environment that supports plotting (Jupyter, Colab, or local Python).
