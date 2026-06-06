# Q25 — "Reliable Routes for Santa": A Travel-Time **Reliability** Network of LA (Oct–Dec 2019)

> **For the implementing agent (Claude Code):** This is the spec for Section 10 ("Define Your Own Task") of ECE 232E Project 4, worth 20% of the project and graded on creativity. Implement it end-to-end in Python (igraph + numpy/scipy/matplotlib), reusing the existing Q9–Q24 pipeline wherever possible. **Every "Expected result" below is a hypothesis stated for guidance only — run the code, then report the REAL computed numbers. If a result contradicts the hypothesis, that is a finding to write up, not an error to hide. Do NOT fabricate or back-fill numbers to match the hypotheses.**

---

## 0. Motivation (write this into the report intro)

In Part 2 we minimised travel **time**. But a gift-delivery operation does not care only about the *average* trip — it cares about *predictability*. A route that is usually 10 min but occasionally 40 min is worse than one that is reliably 15 min, because Santa must hit delivery windows. The Uber Movement monthly-aggregate file contains a column we have ignored in the entire project so far — `standard_deviation_travel_time` — which measures how much each origin–destination travel time fluctuates within the month. We use it to build a **reliability network**, define reliability-aware routing, and study how LA's road network becomes *less predictable* across the 2019 holiday season (October baseline → November Thanksgiving → December Christmas shopping).

This task deliberately (1) uses a data field unused by Q1–Q24, (2) introduces a self-defined graph metric and a mean-risk robust-routing formulation that go beyond the graded parts, and (3) calls back to Part 1 (vine clusters / MST), Part 2 (betweenness bottlenecks, max-flow, the "add 20 edges" exercise) for a coherent arc.

---

## 1. Data & Environment

- **Input 1 (already on disk, reused from Q9):** `los_angeles-censustracts-2019-4-All-MonthlyAggregate.csv`.
  Columns: `sourceid, dstid, month, mean_travel_time, standard_deviation_travel_time, geometric_mean_travel_time, geometric_standard_deviation_travel_time`. The Q4 file contains **three months**: `month ∈ {10, 11, 12}` — this satisfies the project's "≥ 3 months" requirement with **no new download**.
- **Input 2 (reused from Q9):** `los_angeles_censustracts.json` (tract polygon corners → centroids).
- **Language/libs:** Python, `igraph`, `numpy`, `scipy.spatial` (already used for Delaunay in Q14), `matplotlib`. Reuse the exact Q9 graph-cleaning function and the Q14/Q15 helpers; do not re-implement them.
- **Output dir:** put all figures/tables under `Q25_outputs/`. Save every figure as PNG (≥150 dpi) and every table as both CSV and a rendered markdown/HTML block in the report.

### Convention used throughout
For each month `m ∈ {10, 11, 12}` build the cleaned undirected graph exactly as in Q9 (largest connected component; merge A→B and B→A by **averaging** `mean_travel_time`, and combine their std via the variance-of-average rule below). Each edge `e=(i,j)` carries:
- `mu_e`  = mean travel time (seconds) — same quantity as the Q9 weight.
- `sigma_e` = combined std travel time (seconds). When merging the two directed records, use `sigma_e = sqrt((sigma_AB^2 + sigma_BA^2) / 4)` (std of the average of two independent estimates). If only one direction exists, keep its sigma.
- `cv_e` = `sigma_e / mu_e`  (coefficient of variation, unitless) — the **unreliability weight**.

---

## 2. Definitions (state these in the report, with equations)

1. **Edge unreliability** = coefficient of variation `CV_e = sigma_e / mu_e`. Build `G_rel^m` with edge weight `CV_e`, alongside the time graph `G_time^m` with edge weight `mu_e`.
2. **Mean-risk edge cost** (for robust routing), with risk-aversion parameter `k ≥ 0`:
   `c_e(k) = mu_e + k * sigma_e`.
   Routing with this additive cost via Dijkstra yields a **robust shortest path**. Default `k = 1.645` (≈ the one-sided 95th percentile under a normal approximation, i.e. a "95% on-time" buffer).
3. **Path-level statistics** for a path `P` (edges treated as independent within the month — state this assumption explicitly):
   - mean time:      `T(P)  = Σ_{e∈P} mu_e`
   - variance:       `Var(P)= Σ_{e∈P} sigma_e^2`
   - std:            `S(P)  = sqrt(Var(P))`
   - **planning time:** `PT(P, k) = T(P) + k * S(P)`  (the time you must budget to arrive on-time with prob ≈ Φ(k)).
4. **Price of Reliability** for an O–D pair, comparing the time-optimal path `P*` (min `Σ mu`) to the robust path `P_k` (min `Σ c_e(k)`):
   - `PoR = (T(P_k) − T(P*)) / T(P*)`            (extra *average* time you pay)
   - `VarRed = 1 − Var(P_k) / Var(P*)`           (variance you buy back)
   - `PTgain = (PT(P*,k) − PT(P_k,k)) / PT(P*,k)`(reduction in budgeted/planning time)

---

## 3. Sub-tasks

### (a) Build the reliability graph; CV distribution across the season
- For each `m`, build `G_time^m` and `G_rel^m`. Report a table: month → #nodes, #edges, mean `mu`, median speed (reuse Q15 speed = geo-dist/`mu`), mean `CV`, median `CV`, 90th-pct `CV`.
- **Figure A:** overlaid histograms of `CV_e` for Oct/Nov/Dec.
- **Hypothesis (verify):** the `CV` distribution shifts right and its tail thickens Nov→Dec; the interesting claim to test is whether **variance rises faster than the mean** (i.e. `CV` rises even where `mu` barely moves) during the shopping season.

### (b) Reliability-MST vs Time-MST
- For December, compute `MST(G_time)` and `MST(G_rel)`. Report total costs, the highest-degree hub of each, and **edge-set Jaccard overlap**.
- **Figure B:** plot both MSTs on real coordinates; colour edges by membership (in both / only-time / only-reliability).
- **Hypothesis:** the two trees share a backbone but diverge on specific corridors — some *fast* roads are *unreliable* (in only-time MST, not reliability MST) and vice versa. Interpret as: the reliability MST chains tracts by *consistency*, not proximity — a reliability analogue of the Part-1 vine clusters.

### (c) **Price of Reliability** (core result)
- Benchmark O–D set: (i) the Q16 Malibu→Long Beach pair (reuse the same nearest-node lookup), plus (ii) 200 random O–D pairs sampled from the December graph.
- For each pair compute `P*` and `P_k` (k=1.645), then `PoR`, `VarRed`, `PTgain`.
- Report summary stats (mean/median across the 200 pairs) + the single Malibu→Long Beach row in detail.
- **Sensitivity:** sweep `k ∈ {0.5, 1, 1.645, 2, 3}` and plot mean `PoR` (x) vs mean `VarRed` (y) — a **Pareto/tradeoff curve**.
- **Figure C1:** the k-sweep Pareto curve. **Figure C2:** for the Malibu→Long Beach pair, draw `P*` and `P_k` on the map.
- **Hypothesis:** the curve is convex — a small extra mean time (low `PoR`) buys a large `VarRed`, i.e. reliability is cheap at the margin. This is the headline result.

### (d) Critical-and-fragile edges (criticality × fragility)
- On `G_time^m` compute **edge betweenness** (reuse the weighted Brandes from Q24). Define the fragility-weighted criticality score `score_e = betweenness_e * CV_e` (rank-normalise each factor to [0,1] before multiplying so neither dominates by scale; report both raw and normalised).
- Identify the **top-20** edges by `score_e` per month = heavily-used AND unpredictable = the network's worst liabilities.
- **Figure D:** map the top-20 critical-fragile edges for Dec (thickness ∝ betweenness, colour ∝ CV). Report a table tracking how many of Dec's top-20 were already in Oct's/Nov's top-20 (set overlap across the season).
- **Hypothesis:** a small number of arterial/freeway segments are simultaneously central and increasingly unreliable in December → concrete intervention targets.

### (e) Reliability-aware road construction (callback to Q19–Q24)
- Starting from December `G̃_∆`-style trimmed graph (reuse Q17 trimming), add **20 new edges** under a NEW objective that parallels Q22 but on the **planning-time** metric:
  `extra_planning(v,s) = PT_shortest_path(v,s, k=1.645) − euclidean(v,s)/effective_speed(v,s)`
  where `effective_speed` is defined path-level as in Q22. Pick the top-20 pairs by `extra_planning` (static version), add edges, recompute the **average planning-time-adjusted shortest path** over all pairs.
- New-edge attributes: geo length `d = euclidean*69` mi; assign `mu_new = d / v_design` with `v_design` = the network's free-flow speed (use the 95th-pct of per-edge speed, or 35 mph if simpler — state the choice); `sigma_new = 0.10 * mu_new` (a well-designed new road is reliable).
- **Table E:** Baseline vs this reliability strategy vs (re-using Q24 numbers) S4 (travel-time static) and S6 (betweenness), all reported under BOTH `Avg time` and `Avg planning time`.
- **Hypothesis:** targeting reliability bottlenecks reduces the network's 95th-pct (planning-time) path cost more than the geometric/time strategies did, even if it barely moves `Avg dist` — mirroring how Q24's S6 helped `Avg time` the most.

### (f) Seasonal synthesis (the "Q8-style" closer)
- One consolidated table across Oct/Nov/Dec: #nodes, #edges, mean `CV`, reliability-MST cost, #critical-fragile edges, Malibu→LB `PoR` and `PTgain`.
- Narrative: how does the holiday season degrade reliability, which corridors, and what does it mean for Santa's routing (prefer robust paths; the modest `PoR` is worth it).
- **Limitations to state explicitly:** (1) normal approximation behind the `k`-buffer and percentile reading; (2) within-month edge-independence assumption for `Var(P)`; (3) monthly aggregation hides hour-of-day peaks (this file has no hourly breakdown); (4) `CV` from Uber's sampling carries its own estimation noise in low-trip tracts.

---

## 4. Deliverables checklist
- Figures: A (CV hists), B (two MSTs), C1 (Pareto curve), C2 (two routes on map), D (critical-fragile map). All in `Q25_outputs/`.
- Tables: (a) seasonal stats, (c) PoR summary + benchmark row, (d) top-20 overlap, (e) strategy comparison, (f) synthesis. CSV + in-report.
- A self-contained report section "Q25" written in the same voice as the existing report (procedure → answer → interpretation), ~3–5 pages, ending with the limitations paragraph.
- Clean, commented code in a `q25_reliability.py` module that imports/reuses the existing Q9–Q24 helpers (do not duplicate the graph-cleaning logic).

## 5. Implementation pitfalls (read before coding)
- **Robust path routing** uses the *additive* edge cost `mu + k*sigma` in Dijkstra; the path's *true* std is `sqrt(Σ sigma^2)`, NOT `Σ sigma`. Use the additive cost only to *select* the path, then recompute `T(P)`, `Var(P)`, `S(P)`, `PT(P,k)` from the chosen edges. Keep these two clearly separate.
- **Fair month comparison:** node/edge coverage differs per month. For weight-shift comparisons in (a)/(f), also report results restricted to the **edge intersection** across the three months, in addition to each month's full graph.
- **Scale before multiplying** in (d): betweenness and CV live on totally different scales; rank-normalise each to [0,1] first.
- Reuse the Q16 nearest-node lookup verbatim for the Malibu/Long Beach coordinates so (c) is comparable to Part 2.
- Set and report a fixed RNG seed for the 200-pair sample in (c).

## 6. Hard rule
Report measured numbers only. Where a measured result contradicts a hypothesis in this spec, write up the discrepancy and a plausible mechanism — that is exactly the kind of analysis that earns the creativity credit.