# Synthetic Pricing Panels

Reproducible **data-generating processes (DGPs)** for price-elasticity research on weekly
retail panels. Three generators produce a common `SKU × region × week` schema, inject a
**known ground truth**, and deliberately embed identification regimes that a sound
diagnostic layer must be able to *reject* — not just estimate.

This repository contains **only the simulation layer**: no estimator, no decision logic,
no reporting. It is meant to be imported (or copied) as the data source for benchmarking
demand models, causal-inference pipelines, coverage studies and guard/threshold design.

All entity names are fictional and all figures are in generic currency units.

---

## Why simulate

Elasticity estimators are usually validated against panels where nobody knows the answer.
These generators invert that: the elasticity, the pass-through and the regime label live
**inside** the DGP and are never exposed to the estimator, so any method can be scored on
bias, coverage and — most importantly — on whether it correctly **abstains** where the
data cannot identify anything.

---

## The estimand: the truth is `β · ρ`, not `β`

Volume is generated against the **shelf** price, while the lever a producer controls (and
the regressor a demand model sees) is the **list** price:

```
log Q = c + β · log(P_shelf_real / P̄_shelf) + γ · log P_comp + ...
P_shelf_real = P̄_shelf · (P_list_real / p₀)^ρ · (1 − 0.6 · promo_depth)
```

which collapses to

```
log Q = c + (β · ρ) · log(P_list_real / p₀) + ...
                └──┬──┘
                   θ   ← what a model on list price actually estimates
```

So **θ = β · ρ**. `β` is the *consumer* elasticity (against the shelf price) and is only
recovered after deconvolving the pass-through `ρ`. Scoring an estimator against `β` when
it estimates `θ` reports pass-through as bias.

In the `sticky_shelf` regime the shelf price is pinned to a psychological round number,
the list price never enters volume, `ρ = 0` and therefore `θ = 0`. That presentation must
be **rejected** by the diagnostics, not estimated.

---

## Identification regimes

Every regime exists to break identification in a specific, diagnosable way.

| Regime | What the generator does | Expected verdict |
|---|---|---|
| `clean` | Discrete list-price steps (3–7.5%), never on a promotional week | **Identifiable** |
| `weak` | Only two steps, below 1% each | Not enough variation |
| `sticky_shelf` | Shelf price frozen at a round number → `ρ = 0` | Reject: zero pass-through |
| `confounded` | List price moves in lockstep with promotional depth | Price and promo inseparable |
| `collinear` | List price tracks the competitor's price | Own price confounded with competitive response |

Only `clean` is identifiable by construction. A pipeline that returns a confident
elasticity for the other four is reporting noise.

---

## Contents

| Notebook | Generator | Output |
|---|---|---|
| `Synthetic_Sales_Panel_Generator.ipynb` | `make_synthetic_panel` | Baseline panel: 6 presentations × 6 regions × 120 weeks × 8 SKUs → **5,760 × 21** |
| `Synthetic_Panel_Generators_Experimental.ipynb` | `make_lab_panel` | Single-presentation bench with explicit knobs → **720 × 21** |
| `Synthetic_Panel_Generators_Experimental.ipynb` | `make_extended_panel` | Portfolio panel: 8 presentations × 30 regions → **28,800 × 30** (21 observable + 9 truth columns) |

### 1 · Baseline panel — `make_synthetic_panel`

The reference DGP. Six presentations across three brands and two categories, one per
regime family, over six regions and 120 weeks. Includes an inflation drift, a wage index,
a public-holiday calendar, regional temperature seasonality, promotional bursts, cost
shocks, retailer margin dynamics and **forward buying** (stores load up two weeks before
an announced increase above 2%, so sell-in leads sell-out).

### 2 · Lab bench — `make_lab_panel`

One `clean` presentation, every design quantity exposed as an argument, for sweeping a
single dimension at a time (minimum detectable effect, break detection, estimator
comparison).

| Knob | Purpose |
|---|---|
| `n_moves`, `mag` | Number and magnitude of list-price steps — the identification budget |
| `beta`, `pt` | True consumer elasticity and pass-through → `θ = beta · pt` |
| `beta_break=(week, β_post)` | Structural break in the elasticity |
| `comp_moves`, `comp_mag` | Discrete competitor moves |
| `comp_react` | Competitor reaction to our own price, one-week lag |
| `cross` | Cross-price term on `log(P_comp)` |

Two knobs deserve emphasis:

- **`comp_moves`** — with a random walk alone (0.4% weekly sd), the competitor's log price
  varies by ~1.5% over 120 weeks. Against demand noise of 0.05 the cross channel sits
  **below the noise floor**: cross-elasticity is unidentifiable *by construction*. Set
  `comp_moves > 0` before claiming anything about the competitive bracket.
- **`comp_react`** — with `comp_react = 0` the competitor is exogenous, `r̂ ≈ 0`, and the
  identity `θ_total − θ_cond = θ_cross · r` holds trivially. It tests nothing.

### 3 · Extended portfolio — `make_extended_panel`

Fixes six structural limitations of the baseline generator — all of them CPU hours rather
than a quarter of data negotiation:

1. **30 regions instead of 6.** The floor of a permutation p-value drops from 1/C(6,3) to
   1/C(30,8), and a synthetic-control donor pool becomes viable.
2. **Pass-through heterogeneous by region *and* state-dependent** on the retailer's margin
   (transmission rises when the margin falls below target) — exercises Markov-chain and
   variance-attribution machinery.
3. **Regional income component**, so a cash-pressure index stops being one national series
   and its screening test becomes finite.
4. **Injected asymmetry** `eps_up ≠ eps_dn` (Houck decomposition on the shelf price), so an
   asymmetric-elasticity estimator has something real to recover.
5. **Competitor with discrete moves and a reaction function**, lifting cross-elasticity
   above the noise floor.
6. **Liquidity-driven order refusal**, producing an observable extensive margin for
   two-part models.

Key argument for coverage studies:

```python
# Resample the NOISE while holding the DESIGN fixed:
# identical prices, promotions, competitor and temperature.
panels = [make_extended_panel(np.random.default_rng(7), seed_noise=s) for s in range(200)]
```

`seed_noise` draws demand and sell-in noise from a **separate** RNG. This is the only way
to measure coverage **conditional on the design** — the property a within-panel bootstrap
implicitly claims. Dispersion measured across seeds instead mixes in the variance of the
realized design, which is typically the dominant term and makes an interval look far worse
(or better) than it is.

---

## Panel schema

| Column | Meaning |
|---|---|
| `sku_id`, `pres_id`, `brand`, `category` | Hierarchy: `category > brand > presentation > SKU` |
| `region_id`, `date`, `channel` | Geography, ISO week (Monday) and route-to-market |
| `vol_sellin`, `vol_sellout` | Units shipped to the store / sold to the consumer |
| `price_sellin` | List price charged to the store — **the lever**, nominal |
| `price_sellout` | Shelf price faced by the consumer — set by the retailer, nominal |
| `price_competitor` | Competitor shelf price, nominal |
| `cost_var` | Variable cost per unit, nominal |
| `promo_flag`, `promo_intensity` | Promotional week indicator and discount depth |
| `temperature`, `inflation_index`, `wage_index`, `holidays_in_month`, `business_days` | Exogenous demand shifters |
| `producer_margin` | Real gross margin over the list price |

Nominal series are the real series scaled by the inflation index, so levels drift while
relative prices carry the economic signal.

**Truth columns** (`make_extended_panel` only — never feed these to an estimator):
`retailer_margin`, `rho_base`, `rho_realized`, `beta_true`, `eps_up_true`, `eps_dn_true`,
`theta_realized`, `cash_pressure_true`, `regime_true`.

---

## Ground truth

| Function | Notebook | Returns |
|---|---|---|
| `truth_by_presentation()` | Experimental | Injected `β`, `ρ` and `θ = β·ρ` per baseline presentation |
| `theta_true_lab(beta, pt)` | Experimental | `beta * pt` — the lab bench's true list-price elasticity |
| `truth_from_panel(panel)` | Experimental | Truth **read from the generated panel** |
| `truth_from_config()` | Experimental | Truth **intended** by the portfolio definition |

**Score against `truth_from_panel`, not `truth_from_config`.** Once pass-through is
heterogeneous and state-dependent, `ρ` is no longer the number in the config dict: the
estimand is the log-log slope actually realized in the series. Comparing against the
config reports *generator dispersion* as *estimator bias*.

---

## Quickstart

```bash
pip install numpy pandas jupyter
jupyter lab
```

Run either notebook top to bottom. To use a generator from your own code, copy the
generator cell or extract it to a module:

```python
import numpy as np

panel = make_synthetic_panel(np.random.default_rng(7))          # baseline, 5760 × 21

lab = make_lab_panel(np.random.default_rng(7),                  # sweep one dimension
                     n_moves=4, mag=(0.015, 0.030),
                     comp_moves=8, comp_react=0.5)

ext = make_extended_panel(np.random.default_rng(7),             # portfolio, 28800 × 30
                          asym=0.25, refusal=0.6)
truth = truth_from_panel(ext)
```

**Requirements:** Python ≥ 3.9, `numpy`, `pandas`. Nothing else — no estimation libraries,
no I/O, no network. The baseline panel generates in ~0.1 s; the 30-region extended panel in
under a second.

**Reproducibility.** Every generator takes an explicit `np.random.Generator`. Same seed →
byte-identical panel. No global RNG state, no wall-clock dependence.

**Real data.** `make_synthetic_panel` sits behind `load_master_panel(path)`: point
`DATA_PATH` at a Parquet/CSV with the schema above and the same downstream code runs on
real data, with the synthetic panel as the fallback.

---

## Suggested uses

- **Benchmark an estimator** — recover `θ` on `clean` and check the confidence interval
  covers `β·ρ` at the nominal rate.
- **Design abstention rules** — measure how often a diagnostic wrongly clears
  `confounded`, `collinear`, `weak` or `sticky_shelf`.
- **Minimum detectable effect** — sweep `n_moves × mag` on the lab bench and map the
  identification frontier.
- **Conditional coverage** — fix the design, vary `seed_noise`, and separate estimation
  variance from design variance.
- **Break detection** — inject `beta_break` and time the detector.
- **Guard calibration** — tune thresholds against a known truth instead of intuition, and
  measure the **family-wise** error rate of a whole set of guards.

---

## Notes and caveats

- These are **simulations**, not data. They reproduce the failure modes that matter for
  identification (confounding, collinearity, frozen shelf prices, forward buying,
  incomplete pass-through) but not the full messiness of real transactional data —
  stockouts, coverage gaps, mix shifts, hierarchy churn.
- The regime labels are a **teaching device**: real panels do not come tagged, which is
  precisely why the diagnostic layer has to earn its verdict.
- Base prices, margins and volumes are set to plausible orders of magnitude in generic
  currency units. They are not calibrated to any market.

---

## Author

**Pedro Cadahia** — [ORCID 0000-0002-2474-2071](https://orcid.org/0000-0002-2474-2071)

## Citation

If you use this repository or its data generators in academic research, please cite it as:

```bibtex
@software{cadahia2026pricing,
  author    = {Cadahia, Pedro},
  title     = {pricing-experiments-synthetic-panel: Synthetic Panel Data Generator
               for Pricing Experiments},
  year      = {2026},
  version   = {0.1.0},
  publisher = {GitHub},
  url       = {https://github.com/pedrocadahia/pricing-experiments-synthetic-panel},
  orcid     = {https://orcid.org/0000-0002-2474-2071}
}
```

A machine-readable [`CITATION.cff`](CITATION.cff) is included, so GitHub renders a
**"Cite this repository"** button and exports BibTeX or APA directly.

## License

Licensed under the **Apache License, Version 2.0** — see [LICENSE](LICENSE) and
[NOTICE](NOTICE) for details.

```
Copyright 2026 Pedro Cadahia

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
