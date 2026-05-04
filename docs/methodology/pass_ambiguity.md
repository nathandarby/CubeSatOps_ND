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

1. **Currently trying to derive first-principle physics based observable difference normalizations between all elements.**
- Let the set of Orbital Elements be $ O = \{a, i, \Omega, e, \omega, B^{*}, M\} $
- Rank Orbital Elements by their intrinsic discriminatory power (should this be derived quantitatively or qualitatively?: Probably qualitative for now (lack of data))
- Normalization uses $\frac{\Delta(O_i)}{interval(O_i)}$ and cross analyzes this with ground-station observables to be physically meaningful. 
- We will need to separate $ O $ into two subsets: $O_b$ and $O_{nb}$, where $O_b$ and $O_{nb}$ stands for bounded and non-bounded orbital elements, respectively. 
- Qualitative Rankings (strongest to weakest for SSO clusters):
  1. **Δi** — different inclinations produce different ground tracks, 
     different latitude reach, different sky elevation profiles. 
     Changes *where* passes happen.
  2. **ΔΩ** — different RAAN shifts the longitude where passes occur. 
     Changes *where* passes happen, but couples with M and Earth rotation.
  3. **ΔM** — direct pass timing offset at the same ground station. 
     Changes *when* a pass happens, all else equal.
  4. **Δa** — produces orbital period difference, drives pass timing drift 
     over orbits. Changes *when* over time.
  5. **ΔB\*** — drives Δa via differential drag. Discriminating power is 
     "future Δa" — accumulated effect over many refresh cycles.
  6. **Δe**, **Δω** — noise-dominated for near-circular SSO orbits. 
     Negligible discriminating power.

- Bounded vs unbounded sets:
  - $O_b = \{i, \Omega, \omega, M\}$ — angular, naturally bounded.
  - $O_{nb} = \{a, e, B^{*}\}$ — physical or dimensionless, no natural max.
    (e is technically bounded [0, 1] but operationally unbounded for LEO.)

- In file `OAGT_POD.ipynb`, we are using historical data from the transporter-9 launch, starting with one TLE and propagating the orbit, adding perturbations, and trying to infer meaningful physical thresholds per element for observers/operators. 


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