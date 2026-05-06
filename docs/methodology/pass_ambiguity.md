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

# Single Element Analysis 
## Inclination
#### Inclination Tree
1) Layer 1 is geometrical. Start with Compute $|\Delta i|$: 
    - if $|\Delta i| < \theta_i \rightarrow$ inclination ambiguous (proceed to layer 2)
    - if $|\Delta i| \geq \theta_i \rightarrow$ discriminated, stop
    - **Note**: $\theta_i$ is not yet derived 

2) Layer 2 is operational. 
    - Start with compute $\Delta d_\perp = f(|\Delta i|, \text{ } \phi_{\text{GS}}, \text{ }i)$
    - Compare $\Delta d_\perp$ agaisnt $r_{\text{vis}} = f(h, \text{ }\epsilon_\text{min})$, **Check r_vis notes**
    - If $\Delta d_\perp > r_{\text{vis}} \rightarrow$ discriminated by pass occurence, stop
    - If $\Delta d_\perp \leq r_{\text{vis}} \rightarrow $ pass is ambiguous, proceed to: 
    1. Mean Anomaly (time dependence)
    2. Doppler Curve Shape (function of angle approach)
    3. Peak Elevation Difference (function of $\Delta d_\perp$)
    4. Other Discriminating Operations

- **Note**: The visibility circle **can** act as a spatial filter or discriminator parameter. This depends on $\Delta d_{\perp}$. Since $\epsilon_{\text{min}}$ is not a universal constant, it is flexible and is a reflection of the team's requirements.
    1) Antenna Clearance (physical obstructions)
    2) Link-Budget Requirements (attenution increasing near horizon)
    3) Noise Floor (atmospheric noise at low elevation)
    4) Operational preferences (longer passes vs signal losses)
- Sweep paramter $\epsilon_{min} \in \{...\}$ can run ambiguity assessment simultaneously and show the team which threshold converts their ambiguous pair into a discriminated one. 

## RAAN $\Delta \Omega$
#### RAAN Tree 
- Follows the same structure as inclination tree. The only differences are: 
    1. **The displacement formula** scales by cos $\phi$ instead of (tan $\phi$ / tan $i$)
    2. **The sensitivity direction** is east-west instead of north-south
    3. **The threshold $\theta_{\Omega}$** is a different physical anchor, that still needs to be derived like $\theta_i$.

- **Notes**: Because of J2 (Earth's oblateness) there is RAAN drift (precession), thus causing $\frac{d\Omega}{dt} \neq \text{constant}$. Pairwise altitude analysis updates RAAN drift information. 

## Mean Anomaly $\Delta M$
#### Mean Anomaly Tree
- This analysis and decision branching differs from inclination and RAAN because **Mean Anomaly** does not change the orbit geometry at all. It tells you **where** in the orbit the satellite is at epoch. Thus, it does not have any **$\Delta d_\perp$** analog and $ \implies$ straight to operational layer.
- It turns out to be more of a scheduling discriminator under both $\Delta i \text{ and } \Delta \Omega$ rather than a standalone tree. 
- A **useful formula**: $$ \Delta M \implies \Delta  t_{pass} = (\frac{\Delta M}{360^\circ})  \cdot T $$ Compare to pass duration $\rightarrow$ either time spearates passes or doesn't. 

## Altitude (semi-major axis $\Delta a$) 
#### Altitude Tree 
Is more interesting because it is time-dependent and relates to the **Mean Anomaly**. The key mechanism is: $$ \Delta a \rightarrow \Delta T$$ $\text{different orbital periods} \implies \text{ satellites drift apart in mean anomaly over time} \implies \Delta M \text{ grows as a function of }\Delta a \text{ and time elapsed}$ 
- This means that **altitude** doesn't produce a static geometric displacement like **inclination** or **RAAN**, but rather produces a growing $\Delta M$. The two are coupled. 
- $\Delta a$ doesn't need it's own decision tree, but rather feeds into $\Delta M$ as a time-dependent input. It is also important to note that RAAN drift depends on true anomaly which depends on altitude. 
- The cleaner **framing** might be: $$ \Delta M(t) = \Delta M_0 + f(\Delta a, \text{ }t) $$ Where $\Delta M_0$ is the initial mean anomaly difference at TLE epoch and $f(\Delta a, \text{ }t)$ is the accumulated drift from altitude difference. 
- **Notes**: This connects directly to BSTAR analysis, so it is worth cross-analyzing BSTAR, Mean Anomaly, and altitude as a single coupled system. 

## Summarizing $i, \Omega, M, a$
#### What TLE refresh gives for free:
- Fresh $i, \Omega, M$ at epoch - low error accumulation
- SGP4 propagation handles a, drag, J2 corrections internally
- Pass windows, pointing geometry, doppler curves all fall out of propagation directly

#### What the framework adds on top:
- $\Delta i \rightarrow \Delta d_\perp \rightarrow$ spatial discrimination relative to $r_{vis}$
- $\Omega \rightarrow$ longitudinal track separation relative to $r_{vis}$
- $\Delta M \rightarrow$ pass time separation relative to pass duration
- $\Delta a \rightarrow$ easy read from mean motion in TLE line 2, direct altitude discriminator 

#### Where the analytical velocity model earns its place:
- Doppler curve shape prediction - specifically the slope at zero crossing which is a function of $v$ and closest approach geometry
- Pre-pass frequency planning without running full propagation
- Connecting $\Delta B^*$ to long-term drift estimates 

## Eccentricity $e$ and Argument of Perigee ($\omega$)
- Assuming a constant inclination and RAAN, $\Delta e$ only stretches the orbit from/to circular from/to elliptical. This means perigee drops, apogee rises, while the semi-major axis (avg alt) stays constant. The satellite spends more time near apogee (slower) and less near perigee (faster) per Kepler's second law. 
- For near circular LEO rideshare clusters $\Delta e \approx 0.001$ gives a radial variation of $$ \Delta r = a \cdot \Delta e \approx 6800 x 0.001 \approx 7 \text{ km } $$Which is small but not completely negligible and shows up as a slight altitue oscillation once per orbit rather than a constant offset $\Delta a$. 
- **Identification**: Low km radial oscillation produces negligible ground track displacement and essentially no measurable Doppler signature difference compared to $\Delta i, \Delta \Omega, \text{ or } \Delta a$. 
- **Long-term ops**: For a nearly circular LEO CubeSat, $e$ naturally damps towards zero over time via atmospheric drag anyway. Not really a parameter worth monitoring or acting on. Same as $\omega$. 


# Generalized Multi-Element Ambiguity Measure
Single-element analysis was useful in understanding the physics of TLE orbital elements and their pass window/operational implications. However, creating single-element decision trees, and attempting to define an ambiguity measure as a function of the different $\Delta \{i,\Omega, a, M, etc..\}$ is most likely fallible without enough data. 

After considering the physical mechanism teams use to identify and track satellites (antenna beamwidth), a strong candidate for object discrimination and contact prioritization becomes **angular separation**. For pairwise set of satellites, this is intuitive and slightly non-trivial.

Angular separation allows us to define an ambiguity measure that is inherently multi-element, unifying the single-element analysis, and sets up further operational tasks like doppler shift, mean anomaly tracking, visibility circle strategies, peak elevation analysis.

### Core Methodology
- Layer 1: SPG4 + ENZ + $\alpha [k] > \alpha_{beam}$ (geometric, GS-independent)
- Layer 2: Visibility Radius, $\epsilon_{min}$, antenna tier (operational, GS-specific)
- The unified ENZ pipeline replaces individual element trees
- **$\alpha (t)$** as the single discrimination metric: Can be decomposed into azimuth and elevation angle differences (tied to $\Delta i$ and $\Delta \Omega$).
- $\epsilon_{min}$ is a spatial filter connecting to $r_{vis}$ and the visible pass window.This can updated strategically by the team based on their needs. 

# ME-A Framework Derivation 
The SGP4 analytical algorithm uses orbital elements from TLE lines to propagate predicted orbits from TLE epoch. It is useful for predicting the position and velocity of Earth-orbiting objects, and offers a trade-off between speed and accuracy for short-term predictions. It is a standard for CubeSat teams and analyzing pairwise propagations inherently analyzes pairwise TLE orbital element differences. 

Since most CubeSat teams use antenna beamwidth for satellite communications, the primary method to connect a satellite involves angular separation. Angular separation in 3D coordinates inherits information about single-element deltas, and can be decomposed into azimuth and elevation angles. This makes it a strong candidate for short term (fresh TLE) pairwise ambiguity prediction. 

Our goal is to compare $\alpha_{pair}$ to $\alpha_{beam}$ at each timestep to understand short term pairwise satellite separation. SGP4 is a continuous function. However, the discrete nature of plotting SPG4 propagations leads us to approximate $\alpha (t)$ as **$\alpha [k]$**, where $k \in \{1, 2, \cdots, n\}$ and n is the number of time_steps. (timestep-based SGP4)

#### Coordinate Frame Identification 

Assume two satellites are orbiting Earth, passing over a ground station at nearly the same time. What is their angular separation? First we have to ask, angular separation **relative** to where. The trick here is to identify the correct coordinate chain. We will state the logical flow up front: $$ \text{TLE} \rightarrow \text{SGP4} \rightarrow \text{TEME} \rightarrow \text{ITRS} \rightarrow \text{ENZ} $$

#### Math 

We are given indexed TLEs. At each timestep k, we take the TLE and convert it to TEME. $$ \vec{r_i}[k] = (x_i, y_i, z_i)_{\text{TEME}}$$

This is the position vector of the indexed satellite in the TEME coordinate frame. An inertial coordinate system fixed to the stars, with the center of Earth as the origin. 

The Earth is constantly rotating on it's spin axis. To account for this rotation, we convert from TEME to ITRS (frame of reference that rotates with Earth). 
$$ \vec{r_i}^\text{ITRS}[k] = R_{\text{TEME} \rightarrow \text{ITRS}}(t_k) \cdot \vec{r_i}[k]$$ The angle of separation depends on a reference coordinate. The reference is the ground station location, or wherever the antenna beamwidth is. The GS geodetic coords: $(\phi_{GS}, \text{ }\lambda_{GS}, \text{ }h_{GS}) $ gives: $$ x_{GS} = (R_E + h_{GS}) \cos \phi \cos \lambda $$ $$ y_{GS} = (R_E + h_{GS})\cos\phi\sin\lambda$$ $$z_{GS} = (R_E + h_{GS})\sin\phi $$ The 3D cartesian coordinates relative to Earth's center is now $$ \vec{r}_{GS} = (x_{GS},\text{ } y_{GS},\text{ } z_{GS}) $$ We can now take the ITRS position vectors (look vectors) of the indexed satellite relative to the ground station. $$ \vec{p}_i^{\text{ ITRS}}[k] = \vec{r}_i^{\text{ ITRS}}[k] - \vec{r}_{GS}$$ This is what the antenna is actually tracking. For a more intuitive feel, we will want to convert look vectors into ground station ENZ components (East, North, Zenith from ground station). 
$$ 
R_{ENZ} = \begin{pmatrix}
-\sin\lambda & \cos\lambda & 0 \\
-\sin\phi\cos\lambda & -\sin\phi\sin\lambda & \cos\phi \\
\cos\phi\cos\lambda & \cos\phi\sin\lambda & \sin\phi
\end{pmatrix} $$ 
Thus $$ \vec{p}_i^{\text{ ENZ}}[k] = R_{ENZ} \cdot \vec{p}_i[k] = (E_i, N_i, Z_i)[k] $$ A satellite directly above will has E = 0, N = 0, Z = radial distance. An object low on the horizon will have a large E, but a small Z. We have finished setting up the coordinate systems to allow for angular decomposition and analysis. 

Azimuth vs Elevation: $$ Az_i[k] = \arctan 2(E_i[k], N_i[k]) $$ $$ El_i[k] = \arctan (\frac{Z_i [k]}{\sqrt{E_i^2[k] + N_i^2[k]}} )$$ 
**FINISH LATER** A few notes: 
- Azimuth is 0 to 360 degrees, 
- Elevation is .....

After decomposing satellite separation into azimuth and elevation angles. We can find pairwise differences at each time step k: 
$$
\Delta Az_{i,j}[k] = min(\text{ }|Az_i[k] - Az_j[k]|\text{ }, 360^{\circ} - |Az_i[k] - Az_j[k]|\text{ })
$$ 
$$
\Delta El_{i,j}[k] = |El_i[k] - El_j[k]|
$$
We define the **peak discrimination** over a pass window as 
$$ 
\alpha_{max} = \underset{k}{max} [\text{ }max(\Delta Az[k], \text{ }\Delta El[k]) \text{ }]
$$ If $\alpha_{max} < \alpha_{beam}$$ then distinguishability is very difficult to resolve geometrically. 

The **discrimination condition** we define: 
$$ \alpha_{max??} > \alpha_{beam} \text{ : Discrimination Opportunity} $$
$$ \alpha_{max} < \alpha_{beam} \text{ : Proceed to Layer 2} $$

**IS max necessary for discrimination condition?** 












