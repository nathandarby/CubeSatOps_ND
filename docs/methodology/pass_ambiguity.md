# Pass Ambiguity

**Status:** Draft — Layer 1 in active development; Layers 2 and 3 scaffolded.
**Last updated:** April 26, 2026
**Location:** `docs/methodology/pass_ambiguity.md`

---

## Problem

When a CubeSat team's satellite is deployed on a rideshare mission, Space-Track assigns preliminary TLEs to a cluster of unidentified objects in radar detection order. The team faces 50+ candidate TLEs, no clear way to know which is theirs, and limited ground-station time to attempt contact. The cost of a wrong contact attempt isn't just wasted time — it's a failed pass that the team can't tell *was* failed, because they don't know whether they pointed at the wrong satellite or the right satellite at the wrong moment.

**Pass ambiguity** is the operational measure of "how hard is it to tell two candidate satellites apart through passive observation." It's the single most important quantity for guiding contact-attempt scheduling during the identification window, because it tells the operator *which* candidate pairs are genuinely confusable and *which* contact attempts will actually be diagnostic.

Pass ambiguity is not one number. It decomposes into three layers, each answering a different operational question, on different timescales, with different inputs.

---

## Three-layer framework

| Layer | Name | Timescale | Inputs | Question answered |
|---|---|---|---|---|
| 1 | Orbital Ambiguity | Time-invariant (modulo maneuvers) | TLE orbital elements | Can these ever be distinguished by passive geometry? |
| 2 | Observer Ambiguity | Pass-window scale (minutes) | Ground-station geometry, pass predictions | Will tonight's pass discriminate this pair? |
| 3 | Instantaneous Ambiguity | Pre-pass (~1 hour) | Current state vectors | Further operational validation when L1 or L2 flags ambiguity |

The layers are computed independently — none is fit to the others' outputs. Their *agreement* is informative; their *disagreement* surfaces operationally interesting cases (e.g., orbits that are inherently similar but happen to be temporally well-separated tonight).

The layers are also roughly *sequential* in operator workflow. Layer 1 runs once on day 1 of the identification window and gives the team a triage view: which pairs are inherently ambiguous, which are trivially distinguishable, which are somewhere in between. Layer 2 runs nightly to schedule the next round of contact attempts. Layer 3 runs immediately before a pass when L1 or L2 has flagged risk and the operator wants short-horizon confirmation.

---

## Layer 1 — Orbital Ambiguity

### What this layer captures

Orbital ambiguity is the property of two satellites' *orbits themselves* — independent of where the satellites currently are along those orbits, independent of the observer, independent of when in the identification window we look. Two satellites with identical orbital elements have maximum orbital ambiguity regardless of their current positions or anyone's ground station.

This is the right starting point for the framework because it's:

- **Operator-independent** — same answer regardless of where the ground station is.
- **Time-invariant** (modulo maneuvers) — TLE-fitting noise causes day-to-day jitter in element estimates, but the underlying orbit is structurally fixed unless the satellite maneuvers. Per-day tracking of orbital ambiguity therefore reveals maneuvering as drift in an otherwise stable signal.
- **Computable from TLE alone** — no propagation, no observer model, no ground-station data. Fast.

The formal claim: orbital ambiguity is necessary but not sufficient for pass ambiguity. Two satellites with identical orbits can still be operationally distinguishable on a given night if they happen to be at different orbital phases (one over Cal Poly while the other is on the opposite side of Earth). Orbital ambiguity tells the operator whether passive identification is *fundamentally possible*; it does not tell them whether it will work *tonight*. The latter is Layer 2's job.

### What "orbit" means here

A TLE encodes seven elements:

- **Semi-major axis** `a` — orbit size, derived from mean motion. Determines orbital period.
- **Inclination** `i` — orbit plane tilt relative to equator.
- **Right ascension of ascending node** `Ω` — orientation of orbit plane.
- **Eccentricity** `e` — orbit shape (0 = circular).
- **Argument of perigee** `ω` — orientation of orbit ellipse within the plane.
- **Mean anomaly** `M` — *position along the orbit at epoch*. This is **excluded** from Layer 1.
- **BSTAR** `B*` — drag coefficient, included as a seventh quasi-element because differential drag drives long-term orbit divergence.

Mean anomaly is excluded because it's a *phase coordinate*, not a *shape coordinate*. Two satellites can be in the same orbit (identical `a, i, Ω, e, ω`) but at different mean anomalies — one ahead of the other along the same path. Mean anomaly difference is what makes their *passes* time-separated even when the orbits are identical, which is properly a Layer 2 concern. Including mean anomaly in Layer 1 would conflate "same orbit" with "same place at same time," collapsing the layer distinction we deliberately want.

The six elements that go into the headline orbital ambiguity score are therefore: `a, i, Ω, e, ω, B*`.

### Per-element similarity via percentile-rank

For each element `x` and each pair (A, B), compute the absolute difference `|Δx_AB|` from the latest available TLE for each satellite. Across all pair-days in the identification window, the distribution of `|Δx|` values defines the natural scale of variation for that element in this cluster.

The per-element similarity score is:

$$
\text{sim}_x = 1 - \text{percentile\_rank}(|\Delta x|)
$$

where `percentile_rank` is computed across all pair-days in the window. By construction, `sim_x ∈ [0, 1]` and is uniformly distributed across all pair-days.

**Interpretation:** `sim_x = 0.97` means "this pair is more similar in element x than 97% of pairs in this cluster." A pair scoring 0.97 is in the top 3% of similarity — it stands out from the cluster's natural spread.

### Why percentile-rank instead of fixed scales

An earlier formulation used `sim_x = exp(-|Δx| / SCALE_x)` with physically-motivated scales (e.g., `SCALE_a = 1.0 km`). This was rejected for three reasons:

1. **Self-calibration.** Fixed scales bake in assumptions about what counts as "small" before seeing the data. They work for SSO clusters at typical altitudes but fail when transferred to looser clusters, higher altitudes, or different orbit families.
2. **No arbitrary parameter choices.** Percentile-rank has no scale parameter to defend. The reference is the empirical distribution of the cluster itself.
3. **Robust to distribution shape.** Empirically, the per-day distributions of `|Δx|` for SSO rideshares are unimodal but heavy-tailed (log-normal-ish for `a, e, B*`, narrowly concentrated for `i`, multimodal for `Ω`). Percentile-rank handles arbitrary distribution shapes without parametric assumptions.

The empirical check that motivated this: per-day distributions of `|Δx|` for T9 in the day 18 → day 35 window overlap heavily across days for `a, i, e, ω, B*`, with `Ω` showing day-to-day variation but stable structural features. Single-window normalization (one percentile lookup per element across all pair-days) is therefore justified.

### Tradeoffs of percentile-rank, and the physical-threshold layer

Percentile-rank gives *relative* similarity within a cluster. It does not give *absolute* physical distinguishability. The most-similar pair in a loose cluster might still be physically distinguishable; the least-similar pair in a tight cluster might still be physically indistinguishable. The metric is intra-cluster.

This matters because operators ultimately want to know: "Is this pair distinguishable by passive geometry within my identification window?" That is a physical question with a physical answer that doesn't depend on what other pairs look like.

The framework therefore augments percentile-rank with **physical distinguishability thresholds**: per-element values below which two satellites cannot be distinguished by passive geometry within a typical identification window, regardless of cluster context. These thresholds are derived from orbital mechanics:

- `Δa` produces an orbital period difference of approximately `3π · Δa / v`. Over an N-day window with K passes/day, this accumulates to roughly `3π · Δa / v · N · K` of cumulative pass-window timing drift. Set `Δa_threshold` to the value that produces drift at the operator's pass-window timing precision (~30 seconds for typical small-team antennas).
- `Δi` and `ΔΩ` produce slowly accumulating cross-track separation that becomes visible to angular-precision observers after a window-length-dependent time.
- `ΔB*` produces differential decay; `ΔB*_threshold` is set such that altitude divergence over the window exceeds altitude precision.
- `Δe` and `Δω` are typically not discriminating for near-circular SSO orbits.

The combined per-element similarity is then:

$$
\text{element\_similarity}_x = \text{relative\_significance}_x \cdot \text{distinguishability}_x
$$

where `relative_significance` is the percentile-rank score and `distinguishability` is a physically-derived factor that drops to zero when `|Δx|` exceeds the physical threshold (the pair is definitively distinguishable in this dimension regardless of cluster context).

This produces a metric that is both cluster-aware (via percentile-rank) and physically anchored (via thresholds), and that generalizes across missions and clusters.

### Combination across elements

The headline orbital ambiguity score combines per-element similarities multiplicatively:

$$
\text{orbital\_ambiguity} = \left( \prod_{x} \text{element\_similarity}_x \right)^{1/n}
$$

The geometric mean (rather than raw product) is used because raw product compounds aggressively: four element-similarities of 0.7 multiply to 0.24, which suggests the pair is "barely similar" when in fact each element is moderately similar. The geometric mean preserves the operational property that *all* elements must be similar to score high (any one zero kills the score), while keeping the magnitude interpretable.

For the headline score on SSO clusters, the elements included are `{a, i, Ω, B*}`. `e` and `ω` are computed and stored but excluded from the headline because they are noise-dominated for near-circular orbits.

### Mean anomaly as a Layer 1 → Layer 2 bridge

Mean anomaly is excluded from the orbital ambiguity score itself, but it's recorded for each pair as a bridge to Layer 2. A pair with high orbital ambiguity (same orbit) but very different mean anomaly is *passively distinguishable* tonight even though they are inherently ambiguous — their passes are time-separated. A pair with high orbital ambiguity *and* similar mean anomaly is the worst case: they will be in the same place at the same time, and passive geometry alone cannot tell them apart.

The decision tree the metric supports:

- High orbital ambiguity + similar mean anomaly → both satellites overhead simultaneously; passive listen attempt is not diagnostic. Use the pass for active commanding instead, or skip.
- High orbital ambiguity + different mean anomaly → time-separated passes; passive listen *is* diagnostic. Schedule both windows and use timing to discriminate.
- Low orbital ambiguity → orbits will diverge naturally. Pass timing differentiates within days regardless.

### Maneuvering detection

Per-day tracking of orbital ambiguity provides a free maneuvering-detection signal. For non-maneuvering satellites, orbital elements drift only with TLE-fit noise — the per-day orbital ambiguity score for a pair is essentially flat across the window. For maneuvering satellites, elements shift discontinuously (discrete burns) or continuously (electric propulsion), causing the per-day score to drift visibly.

The standard deviation of orbital ambiguity score across the window, computed per pair, serves as a maneuvering-likelihood proxy. Pairs where one satellite shows anomalous element variance compared to the fleet are flagged for elevated uncertainty in the orbital ambiguity claim — the score is computed but the operator should treat it with caution because the underlying orbits may not be stable.

This is the basis for a fifth discriminator (beyond altitude, BSTAR, quality score, and minimum separation) — maneuvering detection — which has independent operational value beyond identification.

---

## Layer 2 — Observer Ambiguity

**Status:** Scaffolded. Detailed development in Section 4 of project work.

Observer ambiguity quantifies whether two satellites are operationally distinguishable from a *specific* ground station during a *specific* pass. It depends on pass-window geometry (timing, peak elevation, duration), Doppler curve similarity, and angular separation in the sky during overlap.

Two satellites in identical orbits but at different mean anomalies have low observer ambiguity (their passes are time-separated). Two satellites in slightly different orbits at the same mean anomaly have high observer ambiguity tonight (overlapping passes, similar Doppler), even though their orbits will diverge over time.

Observer ambiguity is the metric that translates orbital structure into "what should the operator do tonight." It builds on pass prediction over the team's ground station, and is developed in detail in Section 4.

---

## Layer 3 — Instantaneous Ambiguity

**Status:** Scaffolded. Likely a triggered/conditional layer.

Layer 3 is invoked when Layer 1 or Layer 2 flags a pair as ambiguous and the operator wants short-horizon (~1 hour) validation before a contact attempt. It is essentially a refined check: given the pair's current state vectors propagated forward to the pass start, does the Layer 2 prediction still hold?

In practice this layer may be largely redundant with Layer 2 if Layer 2's pass prediction is accurate enough. It is reserved for "further operational validation" cases where pre-pass refinement is needed.

---

## Open questions

- **Cross-mission threshold calibration.** Physical distinguishability thresholds need to be validated against T6, T7, T8, T9 data. The thresholds derived from first-principles orbital mechanics should match empirically observed discrimination outcomes; systematic mismatches indicate model corrections.
- **Operator capability profiles.** Different operators have different effective floors (laser ranger vs hobbyist radio). The framework should support a small set of operator profiles that adjust thresholds. Defaults assume "typical small-team capability."
- **High-eccentricity generalization.** The framework assumes near-circular orbits where ω and e are noise-dominated. For eccentric clusters (GTO rideshares, lunar transfers), ω and e become discriminating and the headline score should include them. Tschauner-Hempel or Yamanaka-Ankersen formalisms may be needed for the related Layer 2 work but not for Layer 1 element comparison.
- **Maneuvering threshold.** The per-pair element-similarity standard deviation that constitutes "anomalous" maneuvering needs cross-mission calibration.
- **Combination of Layer 1 with discriminators.** How does orbital ambiguity interact with the existing four-discriminator framework (altitude, BSTAR, quality score, minimum separation)? Discriminators 1 and 2 are essentially Layer 1 components (altitude is `Δa`, BSTAR is `ΔB*`); the combination should be made explicit rather than parallel.

---

## References within project

- `mission_analysis_test.ipynb` — Generalized analysis pipeline; produces `hist_df` and per-mission element distributions.
- `Orbit_Analysis.ipynb`, Item 13 — Pairwise relative velocity analysis; foundation for Layer 2.
- `HANDOFF.md` — Project state, cross-mission findings (T6-T9), discriminator framework.
- `docs/decisions/2026-04-26_percentile_vs_scale.md` — Decision record for percentile-rank choice (forthcoming).