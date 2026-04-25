# CubeSat Identification Tool — Project Handoff Document
**Author:** Paul (Cal Poly SLO)  
**Date:** April 24, 2026  
**Project Repo:** `cubesat_ND`

---

## 1. Project Overview

### The Problem
When a cubesat launches on a rideshare mission (e.g. SpaceX Transporter series), Space-Track assigns preliminary TLEs to a cluster of unidentified objects labeled sequentially (e.g. `2023-174A`, `2023-174B`...). The piece letters are assigned in radar detection order — not deployment order — so teams have no idea which catalog object is their satellite.

The current identification process requires teams to:
1. Wait for TLEs to appear in the catalog (~8-18 days post-launch)
2. Predict passes for all candidate objects over their ground station
3. Attempt radio contact during each predicted window
4. Match successful contact to the correct TLE

This process is manual, error-prone, and requires orbital mechanics expertise that most university cubesat teams don't have. Analysis across four Transporter missions (T6-T9) shows a consistent **~11 day mean confirmation lag** — not because identification is slow, but because teams lack tools and guidance.

### The Goal
Build an accessible, intelligent identification tool that:
- Requires zero orbital mechanics expertise to operate
- Automatically narrows candidates using physics-based discriminators
- Prioritizes contact attempts for maximum diagnostic value
- Provides plain-language guidance at every step
- Is accessible to small teams with limited budgets

### Target Users
- University cubesat teams (primary)
- Small commercial smallsat operators
- Developing nation space agencies
- Any team without dedicated astrodynamics staff

---

## 2. Repository Structure

```
cubesat_ND/
├── Pass_Predictor.ipynb          # Single satellite pass prediction tool (MVP complete)
├── TLE_Identification.ipynb      # Space-Track API fetcher and TLE pipeline
├── Multi_Pass_Predictor.ipynb    # Multi-TLE pass prediction (in progress)
├── Orbit_Analysis.ipynb          # Deep data analysis on propagated orbits
├── mission_analysis_test.ipynb   # Generalized multi-mission analysis pipeline
├── mission_analysis.py           # Module version (in progress — migrate from notebook)
└── HANDOFF.md                    # This document
```

### Cache Files (stored in `~/mission_cache/`)
```
~/mission_cache/
├── {intldes}_satcat.json         # SATCAT query results (permanent cache)
├── {intldes}_tles.json           # Current TLEs (1hr cache)
└── {intldes}_history.json        # Historical TLEs (permanent cache)
```

---

## 3. Pass_Predictor.ipynb — Current State

### What it does
Single satellite pass prediction tool for ISS over Cal Poly SLO ground station.

### Completed features
1. ✅ TLE fetch from Celestrak by NORAD ID
2. ✅ SGP4 propagation (2000 timesteps over 24hrs)
3. ✅ Ground track — 24hr multi-orbit with spectrum colors
4. ✅ Ground station coverage using elevation angle geometry
5. ✅ Pass predictor — elevation-based with AOS/TCA/LOS
6. ✅ Elevation arc per pass — with peak annotation
7. ✅ TLE age warning
8. ✅ Doppler shift estimator (145.825 MHz ISS APRS)
9. ✅ Pass quality rating (Poor/Fair/Good)

### Key physics implemented
**Elevation angle formula:**
```python
el = np.arctan2(np.cos(sep) - R_e / r_sat, np.sin(sep))
```
Where `sep` is angular separation between ground station and satellite groundtrack point, `R_e` is Earth radius, `r_sat` is orbital radius. This replaces an earlier visibility circle approximation.

**Doppler shift:**
```python
doppler_hz = -f0 * (range_rate / c)
```
Range rate computed via finite difference of range array. Zero crossing at TCA (Time of Closest Approach).

### Ground station parameters
```python
gs_name = "Cal Poly SLO"
gs_lat  = 35.30    # degrees North
gs_lon  = -120.66  # degrees East  
gs_alt  = 0.097    # km
min_elev = 10.0    # degrees minimum elevation
```

### Roadmap (remaining)
- ⏳ Eclipse predictor
- ⏳ Link budget estimator
- ⏳ Multi ground station support
- ⏳ Sun angle / beta angle
- ⏳ Coordinate frame display
- ⏳ Export pass schedule as CSV/.ics

---

## 4. TLE_Identification.ipynb — Current State

### What it does
Fetches and caches TLE data from Space-Track API for a given launch group.

### Key functions implemented

**Login:**
```python
session = requests.Session()
session.post(ST_LOGIN, data={"identity": USERNAME, "password": PASSWORD})
```

**Fetch NORAD IDs (satcat — permanent cache):**
```python
norad_ids = get_norad_ids(intldes, session)
# Queries satcat by INTLDES~~ wildcard, filters OBJECT_TYPE=Payload
# Cached to ~/satcat_cache.json permanently
```

**Fetch current TLEs (gp — 1hr cache):**
```python
tles = fetch_tles(norad_ids, session)
# Queries gp class in JSON format for TLE_LINE0/1/2
# Cached to ~/tle_cache.json for 1 hour
```

**Fetch historical TLEs (gp_history — permanent cache, one-time query):**
```python
historical_tles = fetch_historical_tles(norad_ids, start_date, end_date, session)
# One-time query — never re-query gp_history
# Cached to ~/tle_history_cache.json permanently
```

### Space-Track API notes
- Base URL: `https://www.space-track.org`
- Login endpoint: `/ajaxauth/login`
- Query endpoint: `/basicspacedata/query`
- Rate limits: GP class max 1/hour, SATCAT max 1/day, gp_history one-time
- Use `INTLDES/{designator}~~/` for wildcard matching
- Use `/decay_date/null-val/` to filter to on-orbit objects only
- Use `/format/json` for GP class to get TLE_LINE0/1/2 fields

### International Designator format
```
YYYY-NNNPPP
```
- `YYYY` = launch year
- `NNN` = launch number that year (e.g. 001 = first launch)
- `PPP` = piece letter (A = primary payload, B/C/D... = subsequent objects)

Example: `2023-174A` = first piece of the 174th launch of 2023 (Transporter-9)

**Important:** Don't guess designators — query by launch date:
```python
url = f"{ST_QUERY}/class/satcat/LAUNCH/{date}/OBJECT_TYPE/Payload/format/json"
```

---

## 5. mission_analysis_test.ipynb — The Core Pipeline

This is the main analytical pipeline. Everything is currently in notebook cells — migration to `mission_analysis.py` is in progress.

### Pipeline steps in `run_mission_analysis(intldes, launch_date, session, history_days=60)`

**Step 1 — Fetch NORAD IDs**
Queries SATCAT, caches permanently per mission.

**Step 2 — Fetch historical TLEs**
Queries gp_history for `history_days` post-launch. One-time query.

**Step 3 — Build historical dataframe**
```python
hist_df = build_hist_dataframe(historical_tles, launch_date)
# Adds: epoch (datetime, UTC), date, days_since_launch columns
```

**Step 4 — Confirmation timeline**
```python
confirm_df = compute_confirmation_timeline(historical_tles, launch_date)
# Computes: days_to_catalog, days_to_confirm, confirmation_lag per satellite
# is_confirmed = not starting with 'TBA' or 'OBJECT'
```

**Step 5 — IQR-based altitude clustering**
Takes first TLE per satellite, propagates to get altitude, applies IQR outlier detection:
```python
Q1, Q3 = np.percentile(alt_array, [25, 75])
IQR = Q3 - Q1
lower_bound = Q1 - 1.5 * IQR
upper_bound = Q3 + 1.5 * IQR
core     = satellites within bounds
outliers = satellites outside bounds
```
Core cluster = main deployment group. Outliers = OTV-deployed satellites at different shells.

**Step 5.5 — BSTAR analysis**
For each core satellite:
- Propagate first and last TLE within history window to get alt_first, alt_last
- Compute total_decay = alt_last - alt_first
- Compute decay_rate = total_decay / dt_days
- Extract BSTAR from last TLE line1[53:61] using compressed scientific notation parser

Two correlations computed:
- `corr_daily`: BSTAR vs decay_rate (km/day)
- `corr_total`: BSTAR vs total_decay (km over window)

Filter to decaying objects only (`total_decay < 0`) before correlating.

**Step 5.6 — TLE Propagation Error & Candidate Quality Score**

*Daily sequences:* One TLE per satellite per day (last update of each day).

*Local normalized error:* For each consecutive day pair (n, n+1):
```python
local_error_norm = |position(TLE_n propagated to t_{n+1}) - position(TLE_{n+1})| / dt_days
```
Decomposed into RTN components (Radial, Transverse/Along-Track, Normal).

*Global error:* Propagate TLE_n to day n+k (k=1,5,7,14), compare against actual TLE.

*Predictive power table:* Pearson r between local_error_norm and global error at each k.

*Candidate quality score:*
```python
fleet_median   = median(mean_local_err per satellite)
outlier_thresh = 3 * fleet_median
p95            = 95th percentile of core satellites (non-outliers)
quality_score  = 100 * (1 - min(mean_local_err, p95) / p95)
# Outliers get quality_score = 0.0
```

Flags: `good` / `low` (above P90 of core) / `outlier` (above 3x median)

**Step 5.7 — Pairwise minimum separation**
For each day in first 30 days of history:
- Propagate all core satellites to noon UTC
- Compute all pairwise 3D distances
- Track minimum separation per pair across all days

Output: ranked table of most ambiguous pairs with quality flags.

**Step 6 — RTN separation analysis**
Daily mean pairwise R, T, N components across full history window.
RSI = mean_component(t) / mean_component(t0) — normalized dispersion index.

**Step 7 — Optimal identification window**
```python
# Smooth mean_R with rolling window=5
# Compute gradient (growth rate)
# Find peak after day 10 (avoid early noise)
# Find 50% decay point after peak
# Window = first day of data → 50% decay day
```

**Step 8 — Plot RTN separation**
Two Plotly plots: RSI over time, absolute km over time.

### Return dictionary keys
```python
{
    'intldes', 'launch_date', 'norad_ids',
    'hist_df', 'confirm_df', 'early_alt_df',
    'rtn_df', 'rtn_clean',
    'cluster_focus', 'cluster_norads',
    'peak_day', 'peak_rate', 'inflection_day',
    'bstar_df', 'corr_daily', 'corr_total', 'r2_daily', 'r2_total',
    'daily_err_df', 'sat_quality', 'outliers_q',
    'daily_sep_df', 'pair_df'
}
```

---

## 6. Key Data Findings

### Cross-mission comparison (T6, T7, T8, T9)

| Metric | T6 (2023-001) | T7 (2023-054) | T8 (2023-084) | T9 (2023-174) |
|--------|--------------|--------------|--------------|--------------|
| Launch Date | 2023-01-03 | 2023-04-15 | 2023-06-12 | 2023-11-11 |
| Total Payloads | 99 | 48 | 66 | 95 |
| Hist Satellites | 95 | 47 | 64 | 92 |
| Days TLE Data | 50 | 58 | 51 | 43 |
| Days to Catalog | 14.8 | 8.9 | 11.5 | 19.9 |
| Days to Confirm | 26.1 | 21.2 | 21.8 | 29.2 |
| **Confirm Lag** | **11.3** | **11.6** | **10.3** | **11.0** |
| Core Cluster N | 88 | 36 | 55 | 81 |
| BSTAR total r | -0.568 | -0.581 | -0.165 | -0.445 |
| Prop r day5 | 0.987 | 0.797 | 0.615 | 0.891 |
| Prop r day7 | 0.978 | 0.784 | 0.467 | 0.619 |
| Optimal Window | days 10-35 | days 3-22 | days 8-22 | days 17-31 |
| Min Sep (km) | 0.3 | 0.6 | 0.6 | 0.8 |

### Finding 1 — Confirmation lag is universal (~11 days)
Mean confirmation lag of **11.1 days** across all four missions. This is structural — driven by radar observation cadence + minimum passes needed for first contact. It represents the **baseline pain** the tool addresses.

### Finding 2 — RTN separation: radial dominates
Radial separation (altitude decay driven by differential drag) is the dominant and growing separation mechanism over 60 days. Along-track plateaus after initial deployment timing effect. Cross-track is negligible for SSO rideshares.

Physical mechanism:
- High BSTAR → more drag → slower velocity → lower orbit → higher angular velocity
- This is counterintuitive: drag causes decay to lower orbit where satellite moves faster
- Different BSTAR values → different altitudes → growing radial separation over time

### Finding 3 — Optimal identification window
Peak radial growth rate occurs shortly after first TLE appearance. The 50% decay point (half-peak growth rate) defines the end of the optimal window. Across missions: **days 3-35 post-launch**, with peak discrimination typically around days 15-25.

### Finding 4 — Propagation error predicts future error
Local normalized TLE error (consecutive epoch comparison) predicts global propagation error:
- Day 5: r = 0.615 to 0.987 (mission dependent)
- Day 7: r = 0.467 to 0.978
- Day 14: r = 0.341 to 0.569

This confirms **satellite consistency** — some objects are inherently harder to track due to physical properties (size, tumbling, radar cross-section). Early error is a leading indicator of long-term tracking quality.

### Finding 5 — BSTAR explains ~20-34% of decay variance
After removing maneuvering satellites:
- T6: R² = 0.323
- T7: R² = 0.337
- T8: R² = 0.027 (anomalous — likely maneuvering dominated)
- T9: R² = 0.198

Total 60-day decay correlation is more robust than per-day rate correlation. Filter to `total_decay < 0` before computing.

### Finding 6 — Cataloging speed varies by mission
- T7: 8.9 days (fastest) — earlier cataloging reveals more of the separation curve
- T9: 19.9 days (slowest) — objects took longer to appear in catalog
- Faster cataloging → longer observable window → better discrimination opportunity

---

## 7. The Four Discriminators

### Discriminator 1 — Altitude Filter (Day 0, before TLEs)
**Input:** Team's expected deployment altitude from mission design docs  
**Method:** IQR-based clustering of first TLE altitudes  
**Effect:** Eliminates OTV-deployed satellites, decaying objects, wrong orbital shells  
**Typical reduction:** 47-99 → 36-88 candidates  
**Confidence:** High — altitude differences are large and unambiguous  

### Discriminator 2 — BSTAR Coefficient (Days 3-14)
**Input:** Satellite mass and dimensions → ballistic coefficient  
**Method:** Compare expected BSTAR against candidate BSTAR values  
**Effect:** Probabilistic narrowing based on physical plausibility  
**Typical R²:** 0.20-0.34 (explains 20-34% of decay variance)  
**Confidence:** Moderate — useful pre-filter, not definitive  
**Caveat:** Works best when core cluster has altitude spread (>20km)  

### Discriminator 3 — Propagation Error Quality Score (Days 1-2 after first TLE)
**Input:** Two consecutive TLE epochs per candidate  
**Method:** local_error_norm = |propagated - actual| / dt_days  
**Effect:** Ranks candidates by TLE reliability → prioritizes contact attempts  
**Predictive power:** r=0.60-0.99 for 5-7 day horizon  
**Flags:** good / low / outlier  
**Confidence:** High for prioritization, not for elimination  
**Key insight:** High quality score = contact attempt is diagnostic. Low quality = ambiguous result.  

### Discriminator 4 — Minimum Separation (Ongoing)
**Input:** Daily propagated positions for all core candidates  
**Method:** Pairwise 3D distance computation over identification window  
**Effect:** Identifies most ambiguous pairs — overlapping pass windows  
**Key thresholds:** <100km = highly ambiguous, >1000km = clearly distinguishable  
**Typical pairs <100km:** 62-136 per mission  
**Confidence:** Position-dependent — quality score of pair affects reliability  

---

## 8. Product Vision

### Core value proposition
**"Make satellite operations accessible to every team, regardless of experience or budget."**

Not "identify your satellite faster" — the 11-day lag is structural. The real pain is:
- Extreme stress during the most critical mission phase
- No guidance on where to invest limited ground station time
- Failed contact attempts with no way to know if it's the wrong satellite or wrong predicted window
- Doppler correction not set up → missed signals even when pointed correctly
- Paralysis from 50+ candidates with no prioritization framework

### Product arc

**MVP — Discrimination pipeline:**
- Altitude filter + BSTAR ranking + quality scoring + contact prioritization
- Doppler correction curves per candidate
- Contact attempt logging
- Plain language LLM-powered explanations at every step

**V2 — Probabilistic identification:**
- Bayesian probability update with each contact attempt
- Confidence score per candidate updated in real time
- ML model trained on historical confirmed identifications
- Physical specs input refines BSTAR matching

**V3 — Automated identification:**
- Full pipeline runs autonomously
- Alerts team when confidence threshold reached
- Generates identification report for Space-Track submission
- Dataset grows with every confirmed identification

### ML data strategy
The MVP tool is the data collection mechanism. Every confirmed identification with physical specs contributes:
- NORAD ID, physical dimensions, mass, form factor
- TLE evolution over identification window
- Contact attempt outcomes
- Confirmation timeline

This builds the dataset for V2's ballistic coefficient ML model — which doesn't exist anywhere publicly today.

---

## 9. Key Algorithmic Details

### SGP4 propagation
Using `sgp4` Python library with `Satrec` and `SatrecArray` for vectorized propagation.

```python
from sgp4.api import Satrec, SatrecArray
sats = SatrecArray([Satrec.twoline2rv(t['line1'], t['line2']) for t in tles])
e, r_teme, v_teme = sats.sgp4(jd1_array, jd2_array)
# Shape: (n_satellites, n_timesteps, 3)
```

Julian date split into jd1 + jd2 for floating point precision.

### TEME → ITRS coordinate transform
```python
from astropy.coordinates import TEME, ITRS, EarthLocation, CartesianRepresentation
r = CartesianRepresentation(r_teme[i,j] * u.km)
teme = TEME(r, obstime=times[j])
itrs = teme.transform_to(ITRS(obstime=times[j]))
loc = EarthLocation(*itrs.cartesian.xyz)
lat, lon, alt = loc.lat.deg, loc.lon.deg, loc.height.to(u.km).value
```

### BSTAR parsing
TLE compressed scientific notation: `44929-3` means `0.44929 × 10⁻³`
```python
def parse_bstar(s):
    s = s.strip()
    for i in range(1, len(s)):
        if s[i] in '+-':
            return float('0.' + s[:i]) * (10 ** int(s[i:]))
    return float(s)
```

### RTN frame decomposition
```python
R_hat = r_vec / np.linalg.norm(r_vec)           # radial (toward space)
N_hat = np.cross(r_vec, v_vec)
N_hat = N_hat / np.linalg.norm(N_hat)           # normal (orbit plane)
T_hat = np.cross(N_hat, R_hat)                  # transverse (along-track)

dr = r_prop - r_actual
radial      = abs(np.dot(dr, R_hat))
along_track = abs(np.dot(dr, T_hat))
cross_track = abs(np.dot(dr, N_hat))
```

### Dynamic altitude clustering (IQR method)
```python
Q1, Q3 = np.percentile(alt_array, [25, 75])
IQR = Q3 - Q1
core     = alts within [Q1 - 1.5*IQR, Q3 + 1.5*IQR]
outliers = alts outside this range
```

### Optimal window detection
```python
# Smooth radial separation with rolling mean (window=5)
rtn_clean['mean_R_smooth'] = rtn_clean['mean_R'].rolling(5, center=True, min_periods=1).mean()
# Gradient after day 10 only
rtn_post10 = rtn_clean[rtn_clean['days_since_launch'] >= 10]
peak_idx = rtn_post10['radial_growth_rate'].idxmax()
# 50% decay after peak only
after_peak = rtn_clean[rtn_clean['days_since_launch'] > peak_day]
inflection = after_peak[after_peak['radial_growth_rate'] < peak_rate * 0.5].iloc[0]
```

---

## 10. Open Questions & Next Steps

### Immediate next steps
1. **Run Multi_Pass_Predictor.ipynb** — multi-TLE pass prediction for all candidates simultaneously over Cal Poly. This is the core identification interface.
2. **Build contact logging** — record pass attempts (contact/no contact) per candidate per pass window
3. **Integrate discriminators** — combine altitude filter + BSTAR + quality score into unified candidate ranking
4. **Add LLM explanations** — Claude API integration for plain-language explanations of every output

### Open questions
- Why is T8 BSTAR correlation so weak (-0.165)? Likely formation flying satellites dominating — worth investigating
- Should the quality score be normalized per-mission or globally? Currently per-mission
- How to handle the case where team's satellite IS a maneuvering platform?
- What's the minimum cluster size for reliable BSTAR and quality score analysis? (~30 seems to be threshold)

### Technical debt
- `mission_analysis.py` module incomplete — all functions still in notebook cells
- `resolve_name()` function defined inside loop — should be standalone
- Cache file naming uses old `~/` paths for T9 analysis files, new `~/mission_cache/` for generalized pipeline — needs consolidation
- No error handling if Space-Track session expires mid-run

### Validated missions
- T6: `2023-001`, launch `2023-01-03`
- T7: `2023-054`, launch `2023-04-15`
- T8: `2023-084`, launch `2023-06-12`
- T9: `2023-174`, launch `2023-11-11`

---

## 11. Development Principles

- **Incremental, non-destructive development** — always add new cells, never modify prior ones
- **Theory alongside implementation** — understand the physics before coding
- **Configuration-driven** — parameters at top of notebook, not buried in functions
- **Cache everything** — respect Space-Track API limits, never re-query unnecessarily
- **Verify before generalizing** — test each function standalone before integrating into pipeline
