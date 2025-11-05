# ODD Protocol — Behavioral Epidemic ABM with Social Norm Dynamics

*(Overview–Design concepts–Details; aligned with Grimm et al. 2006/2010 and the 2020 update)*

## 1. Purpose and patterns

**Purpose.** Study how **social norms** (descriptive & injunctive) coevolve with **vaccination decisions** and an **event-driven SIR** process on a two-layer network, and how this coupling shapes long-run **vaccination coverage** and **outbreak size**. The model compares norm-driven behavior to imitation/learning-only baselines and explores **external normative nudges**.

**Target patterns / evaluation criteria.**
- Aggregate **vaccination coverage** trajectories across seasons.
- Aggregate **outbreak size** per season (fraction infected).
- Sensitivity of steady states to **norm weights**, **adaptation**, and **topology** (ER vs. small-world + Klimek–Thurner).
- Relative influence of **injunctive** vs **descriptive** norms over repeated decisions.

These patterns are consistent with the accompanying abstract and model narrative emphasizing multi-season behavior–disease feedbacks and the stronger long-run role of injunctive norms in repeated contexts.

## 2. Entities, state variables, and scales

**Entities.**
- **Agents (i=1..N)**: persons located on two networks (physical & social).  
  - **Vaccination choice** `ndVac[i] ∈ {1(unvacc.),2(vacc.)}`  
  - **Choice memory** `Choicehist2[i][1..mem]` (recent vaccination decisions)  
  - **Payoff memory** `Payoffhist2[i][1..mem]` (recent material payoffs)  
  - **Norm states** in `[0,1]`: `attiVac[i]` (own attitude), `attiAvVac[i]` (injunctive expectation), `prefAvVac[i]` (descriptive proxy)  
  - **Preference to vaccinate** `prefVac[i] ∈ [0,1]` (updated each season)
- **Disease process on physical layer**: event-driven SIR with states per season simulation  
  - `1=S` susceptible, `2=I` infected, `3=R` recovered, `4` vaccinated-exposed (attempted infection on vaccinated)

**Networks.**
- **Layer 1 (physical contacts)**: Either ER with mean degree `k1mean` and giant component ≥ `minint*N`, or Watts–Strogatz small-world (`k`, `betaS`).
- **Layer 2 (social)**: Either copy of layer 1 plus extra random links (mean increase `k2meanextra`), or Klimek–Thurner-like evolving graph with **triadic closure** `r`, **turnover** `p`, and **overlap bias** `c` toward layer-1 neighbors.

**Scales.**
- **Time:** discrete **seasons** (outer loop). Within each season, **Nsim** continuous-time SIR simulations are run to estimate risks.
- **Space:** networks are non-spatial graphs; no coordinates.

## 3. Process overview and scheduling

Per **season** *t*:
1. **Record last decision** into `Choicehist2` (memory length `mem`).  
2. **Seasonal epidemics:** Run `Nsim` event-driven SIR realizations on Layer 1 using current `ndVac` to estimate, for each agent,  
   - infection probability if unvaccinated `infrate[i]` and a neighbor exposure proxy if vaccinated `infneighrate[i]`, and the **mean outbreak size** `outsize`.
3. **Decision update:** For each agent *i*  
   a. Compute **safety** in `[0,1]` by mixing global prevalence and individual (or neighbor) risks.  
   b. Build **empirical preference** from payoff memories (vaccination cost `CV` vs expected infection cost `CI`) via a logistic transform.  
   c. Compute **norm weights** `phiVac` (neighbors’ stability) from recent choice histories and `cons` (neighbors' consensus) 
   d. Combine **empirical** and **normative** channels (own attitude and injunctive/descriptive components), blended by `phiVac`, `cons`, and **safety**, to obtain `prefVac[i]`.  
   e. **Sample** a binary vaccination choice from `prefVac[i]` (Bernoulli). Update vaccinated-neighbor counts incrementally.
4. **Norm update:** Update `attiVac`, `attiAvVac`, `prefAvVac` using `att_update` with social pressure (fraction unvaccinated in ego+neighbors), `phiVac`, `cons`, and external signal `GVac`.
5. **Early stop:** If both infection and vaccination time series are approximately stationary over the last 10 seasons (within 5%), stop; else continue until `vacCycles`.

## 4. Design concepts

- **Basic principles.** Coevolution of **disease transmission** and **behavioral decisions** with **social norms** influencing intentions beyond imitation/learning.
- **Emergence.** Population-level vaccination coverage and outbreak sizes emerge from local risk, memory, and norm interactions.
- **Adaptation.** Agents adjust `prefVac` from payoffs (learning) and norms; adaptation rates `adap` (and derivatives) control speed.
- **Objectives.** Implicitly maximize expected payoff (avoid infection cost `CI`, avoid vaccination cost `CV`) subject to normative pulls.
- **Learning.** Finite-memory, recency-weighted payoff aggregation; logistic mapping from payoff advantage to preference.
- **Prediction.** No explicit forward prediction; risk is **empirically** estimated via Monte Carlo SIR each season.
- **Sensing.** Agents observe neighbors’ recent choices (for `phiVac` and `cons`) and their own/neighbor risks via simulation outputs.
- **Interaction.** Through the two networks: physical (infection) and social (norm pressure; vaccinated-neighbor counts).
- **Stochasticity.** Network construction (ER/rewiring), infection/recovery event times, season seeding of infection, and decision sampling from `prefVac`.
- **Collectives.** No explicit groups; neighborhoods on Layer 2 function as local collectivities.
- **Observation.** Outputs: time series of coverage `vaccomp`, outbreak size `infcomp`, and final `InfoMatrix` summary.

## 5. Initialization

- Networks are generated per `networkOption`. Physical layer must have a giant component ≥ `minint*N` (ER case). Social layer duplicates physical neighbors then adds extra links (Option 1) or uses the Klimek–Thurner process (Option 2).
- `ndVac` initialized by shuffling all-ones (unvaccinated); hence an initially unvaccinated population.
- Choice histories and payoff histories are seeded with random entries (choices in `{1,2}`, payoffs negative randoms) of length `mem`.
- Norm state variables (`attiVac`, `attiAvVac`, `prefAvVac`) initialized i.i.d. `Uniform(0,1)`.
- Random seed recorded in `Params_*` for reproducibility.

## 6. Input data

- No external datasets required. All inputs are **synthetic**: networks, initial states, and random seeds.

## 7. Submodels

- **Event-driven SIR (`event_driven`)**: Continuous-time infection and recovery via exponential clocks on Layer 1. Vaccinated targets convert to state `4` on attempted infection (counted for exposure, not as infections).
- **Season simulator (`Season_Sim`)**: Runs `Nsim` realizations; estimates per-agent risk measures and updates payoff memories.
- **Norm similarity (`phi_Vac`)**: Compares neighbors’ recent choice distributions with their latest choice to measure local stability/consistency.
- **Attitude update (`att_update`)**: Moves `attiVac`, `attiAvVac`, `prefAvVac` toward a mixture of empirical preference and social pressure/authority `GVac`, weighted by `phiVac` and adaptation rates.
- **Preference formation (main loop)**: Recency-weighted payoffs → logistic preference; combined with norms and safety → `prefVac[i]`; sampled to update `ndVac`.

## Model use and experiments

Typical experiments vary: topology (ER vs small-world+KT), norm weights (`gammasocial1..3`), adaptation rates, costs (`CI`, `CV`), and exogenous `GVac`, and compare steady-state coverage/outbreaks and convergence speed.

## Implementation notes

- All arrays and logic reflect the provided code; no functional changes have been made in the commented version.
- Early stopping uses a heuristic within-series coefficient-of-variation threshold (5%) over a 10-season window.

## References (indicative)

- Grimm, V., et al. (2006, 2010, 2020 update). ODD Protocol for ABMs.  
- Model narrative and aims summarized in the companion abstract “Social norm dynamics in a behavioral epidemic model.”
