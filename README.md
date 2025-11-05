
# Behavioral Epidemic ABM with Social Norm Dynamics — README

This repository contains a Julia implementation of a two-layer behavioral epidemic model where **vaccination decisions** coevolve with **social norms**, coupled to **event-driven SIR** dynamics on a physical contact network.

## Files

- **`model_commented.jl`** — The **original model code**, unchanged in logic, with **extensive inline comments** explaining each section, variable, and algorithmic step.
- **`InfoMatrix_*.csv`** — A one-row CSV produced at the end of a run with:  
  `[final_outbreak_size, final_vaccination_rate, mean_prefVac, mean_attiVac, mean_attiAvVac, mean_prefAvVac]`
- **`Params_*.csv`** — The parameter vector logged at the start of the run (includes RNG seed).

## Quick Start

1. Ensure Julia and the listed packages are installed (LightGraphs, Random, StatsBase, Distributions, CSV, DataFrames, etc.).
2. Run:
   ```bash
   julia model_commented.jl
   ```
3. Outputs are written to `InfoMatrix_*.csv` and `Params_*.csv`. Runtime prints the total minutes.

## Model at a Glance

- **Population:** `N` agents (default **100**).
- **Layers:**
  - **Physical (disease spread):**  
    - Option 1 (`networkOption=1`): ER network with mean degree `k1mean`, enforcing a giant component ≥ `minint*N`.
    - Option 2: Watts–Strogatz small-world (`k`, `betaS`).
  - **Social (behavioral influence):**  
    - Option 1: Starts as the physical layer + extra random links (`k2meanextra`).
    - Option 2: Klimek–Thurner-like growth process with triadic closure `r`, turnover `p`, and overlap preference `c` to the physical layer.

- **Epidemics:** Event-driven **SIR** with rates `beta` (infection) and `mu` (recovery). For each season, the model runs `Nsim` outbreak simulations to estimate risk for each agent.

- **Decision-making:** Each season, agents update a **probability of vaccinating** `prefVac[i] ∈ [0,1]` informed by:
  - **Material payoffs** (infection cost `CI` vs. vaccination cost `CV`) estimated from the simulated risks and recency-weighted by a **safety** factor: if vaccinated `safety = min(1 - infneighrate[i], 1)`, else `safety = min(1 - infrate[i], 1)`.
  - **Social norms** (own attitude `attiVac`, injunctive expectation `attiAvVac`, descriptive proxy `prefAvVac`), blended via a **social stability** measure `phiVac` and a **local consensus** factor `Cons`, combined as `φ̃ = sqrt(phiVac * Cons)`.
  - Past choices and payoffs are stored with **finite memory** `mem`.

- **Stopping:** Up to `vacCycles` seasons, or early-stop when both infection and vaccination time series stabilize (last 10 values within 5% of their mean).

## Code Map

- **Network construction**
  - `main_component(g)`: largest connected component (physical layer quality control).
  - `small_world_network`, `klimek_thurner_network`: alternative generators for layered networks.
- **Epidemics**
  - `event_driven(mu, beta, listneigh1, ndVac, ndDiseaseInit)`: event-driven SIR on the physical layer with vaccinated protection.
  - `Season_Sim(...)`: runs `Nsim` outbreaks; returns outbreak size, per-agent infection probabilities, and updates payoff memories.
- **Norms & Preferences**
  - `phi_Vac(...)`: computes stability/consistency of neighbors’ recent choices vs. their last choice (social φ).
  - `Consensus(...)`: computes centered-and-scaled local consensus in ego+neighbors to weight normative influence.
  - `att_update(...)`: updates attitudes and injunctive/descriptive components under social pressure and external signal `GVac`.
  - Main loop: builds `prefVac` from payoff differences (via a logistic transform), `φ̃`, and safety; then samples binary vaccination choices.
- **Outputs**
  - `InfoMatrix_*.csv` and runtime summary.

## Key Parameters

- **Disease:** `beta`, `mu`
- **Economics:** `CI` (infection cost), `CV` (vaccination cost)
- **Memory:** `mem` (used for choices and payoffs)
- **Seasons:** `vacCycles`, **Monte Carlo per season:** `Nsim`
- **Social:** `GVac` (external normative push), `gammasocial1..3` (mixing weights), `adap` (adaptation rate)
- **Topology:** `networkOption`, `k1mean`, `k`, `betaS`, `r`, `p`, `m`, `c`

## Notes & Assumptions

- Vaccinated individuals cannot be infected; attempted infections mark state `4` (used only for exposure accounting).
- The **Consensus** factor is computed from the fraction vaccinated in ego+neighbors, centered around 0.5 and scaled to `[0,1]`, so strong local agreement (either pro- or anti-vaccination) increases normative weight.
- Early-stop checks for stationarity are heuristic.
- Random seed is generated per run and recorded in `Params_*.csv` for reproducibility.

## Reproducibility

The exact parameter vector (including the RNG seed) is saved to `Params_*.csv`. Use it to replicate a run. The model’s internal logic is unchanged; all additions are comments only.
