# Controller I/O contracts — exact inputs and outputs, per mode

This is the reference for "what would it actually take to swap this for a real building."
Every controller mode is a pure function of a small, explicit set of inputs, producing a small,
explicit set of outputs. If you're integrating this against real hardware later, this file is the
seam: replace the *inputs* with a real feed and the *outputs* with a real actuator command, and the
control logic itself does not need to change.

Code: `simulation/cold_storage_simulation.html` — `stepCompressor()` (all modes), `rackRelayStep()`,
`rackContinuousStep()`, `aiPlanZone()` / `aiRollout()` (MPC, AI), `learnerObserve()` (AI).

## Controller I/O reference — current, authoritative

Everything below is per chamber `i` (A Receiving, B Bulk, C Deep freeze); each chamber has its own
controller instance, product, setpoint and compressor rack. Later sections in this file are the
design history; where they disagree with this section, this section is current.

### What the algorithms control (outputs)

| Controller | Decision it makes | How often | Turned into actuator commands by |
|---|---|---|---|
| Rule-based (Two-position) | **How many fixed-speed compressors run** (0…n) — a step thermostat | every 1 min | `rackRelayStep`: picks *which* units (lead-lag by run-hours), contactor on/off per unit |
| Adaptive band | Same stage count, **plus its own setpoint shift** (−0.3·band cheap hours, +0.35·band peak) **and differential width** | every 1 min | same as Rule-based |
| PID (VFD) | **Capacity demand 0–100 % of the chamber's plant** (PI output) | every 1 min | `rackContinuousStep`: VFD lead speed 25–100 % + number of fixed lag units on |
| MPC | **Capacity demand 0–100 %**, the first step of a 12-hour plan | re-planned every 5 min | same rack logic as PID |
| AI (self-learning) | Same as MPC | re-planned every 5 min | same rack logic as PID |

**Not controlled by any algorithm** (fixed plant logic, identical for all five): defrost method and
timing (chosen by coil temperature, every 6-8 h), condenser fans (staged with compressors on relay
racks, floating with load on VFD racks), evaporator fans (continuous / EC speed follows load),
anti-short-cycle timers (5 min run, 3 min rest, ≤6 starts/h per unit), the 55 °C high-pressure
safety trip, lights, doors, deliveries and put-away.

So the single manipulated variable is **cooling capacity per chamber** — a stage count (relay racks)
or a capacity fraction (VFD-lead racks). Everything else follows from it.

### What the algorithms read (inputs)

| Input (per chamber unless noted) | Real-world source | Rule | Adaptive | PID | MPC | AI |
|---|---|:-:|:-:|:-:|:-:|:-:|
| Return-air temperature — lagged ~4 min, ±0.3 °C offset, noise (`measuredAirC`) | Duct/wall RTD on the BMS | ✔ | ✔ | ✔ | | |
| Product-core temperature — ±0.2 °C offset, noise, 15-min filter (`measuredProductC`) | Pulp/core probe | | | | ✔ | ✔ |
| Product target and band | Configured per chamber | ✔ | ✔ | ✔ | ✔ | ✔ |
| Air-setpoint trim = respiration ÷ air-product coupling (≤ ½ band) | Commissioning setting | ✔ | ✔ | ✔ | | |
| Outdoor temperature and humidity | Weather station at the condenser | | | | ✔ | ✔ |
| Room humidity, coil frost level | RH sensor, coil ΔP / frost sensor | | | | ✔ | ✔ |
| Time of day → tariff period | DISCOM ToD schedule | | ✔ | | ✔ | ✔ |
| Demand charge, this month's MD so far, contract demand | Trivector meter / tariff order | | | | ✔ | ✔ |
| Recent and day-by-hour duty cycle | Controller's own history | | ✔ | | | |
| Shift roster, booked deliveries (feed-forward) | WMS / dock booking | | | ✔ | ✔ | ✔ |
| Defrost schedule, put-away schedule, direct deliveries | Plant controller / WMS | | | | ✔ | ✔ |
| Other chambers' planned power (to avoid peaking together) | Supervisory controller | | | | ✔ | ✔ |
| Seasonal ambient curve, clear-sky roof solar | Climate normals / solar model | | | | ✔ | ✔ |
| Nameplate UA, capacity, respiration, COP model | Drawings / datasheets | | | (PI tuning) | ✔ | ✔ (starting point) |
| **Learned:** extra heat, insulation correction, hour-of-day heat, delivery share, put-away delay, weather offset, per-chamber margin | Online learning from the inputs above (`learnerObserve`) | | | | | ✔ |

**Never read by any controller:** true temperatures without sensor error, the hidden as-built
deviations (insulation, capacity, respiration), true equipment wear, the day's actual weather before
it happens.

### What each algorithm optimises

- **Rule / Adaptive:** keep air inside the thermostat differential (Adaptive also shifts the
  setpoint by tariff period). No cost model.
- **PID:** drive air-temperature error to zero (lambda-tuned PI + feed-forward). No cost model.
- **MPC / AI:** minimise over 12 hours: ₹ energy at ToD rates + ₹ increase in the monthly
  maximum-demand charge + ₹ compressor restarts + move-suppression cost + penalty × °C·min of
  product outside its band (priced at shelf-life value while delivered stock pulls down) + penalty
  for supply air too cold − value of cold stored at the end. AI uses its learned model in this
  calculation; MPC uses nameplate values.

### What is measured as the result (benchmark outputs)

Total bill (energy + demand), kWh, peak kVA, % of time each chamber's **true** product core is inside
its band, °C·min outside band, compressor starts, shelf-life consumed — see "Current benchmark".

---

## Historical sections

Everything below records how each mode evolved and why. Inputs/outputs in older sections (e.g. one
compressor per chamber, exact air temperature, one product for all chambers) are superseded by the
reference above.

## Mode A — Two-position (on/off relay), `cfg.mode === "rule"`

**Inputs used:** `airTemp[i]`, `target`, `band`, plus two internal timers (`minRunRemain[i]`,
`minRestRemain[i]`) that only depend on elapsed time since the compressor's own last transition —
no external input.

> **Superseded in v3** — see "Staged racks & realism pass (v3)": thermostat now centred on an
> air setpoint (±½·band, trimmed for respiration), staged over the chamber's rack. Kept for history.

**Control law:**
```
over = airTemp[i] - target
if compressor OFF and over > band and minRestRemain <= 0:  → turn ON, start MIN_RUN_MIN=3min timer
if compressor ON  and over < -band*0.3 and minRunRemain <= 0: → turn OFF, start MIN_REST_MIN=2min timer
```

**Output:** `compressorOn[i]` ∈ {true,false}; `capacityFraction[i]` = 1 or 0 (mirrors `compressorOn`).
Each OFF→ON transition also books a one-time inrush energy charge (~5.8× rated draw for 2s) to the
current tariff period's cost — a real, if small, ₹ cost of cycling, not just a logged event. It also
starts a `startupPenaltyRemain[i] = MIN_RUN_MIN` timer; while it's counting down, `stepZone()`
applies `STARTUP_COP_PENALTY = 0.82` to delivered COP — a cold-start efficiency loss (refrigerant
migration / pressure re-equalization after a stop) that on/off cycling genuinely incurs and smooth
modulation (Mode C) essentially doesn't. This is what actually makes cycling cost more energy than
running continuously at partial load, not an assumption baked one-sidedly into Mode C instead.

**Real-world equivalent:** a mechanical/electromechanical thermostat or a basic PLC relay rung with
anti-short-cycle timers — exactly what the majority of India's bulk potato/onion cold storage runs
today. To connect to a real one of these: read the real air-temp sensor value in place of
`airTemp[i]`, and instead of setting a JS boolean, write to the real contactor's digital output.

## Mode B — Adaptive band, `cfg.mode === "adaptive"`

**Inputs used:** everything Mode A uses, **plus**:
- `lastHourOnMinutes[i]` (rolling 60-minute on/off history) → recent duty cycle
- current tariff period key (`tariffPeriodAt(hour).key`)

**Control law:** same relay logic as Mode A, but `band` is retuned every tick before use:
```
band = base_band
band *= (recentDuty > 0.55) ? 0.7 : 1.25        // load-based: tighten under heavy load, loosen when idle
band *= (period == "peak") ? 1.9                 // tariff-based: loosen during Peak (coast on thermal mass)
        : (period in {"offpeak","solar"}) ? 0.55 // tighten during cheap hours (bank cooling ahead of Peak)
        : 1.0
band = clamp(band, base_band*0.5, base_band*1.5)  // §4d hard safety clamp
```
(An initial ×1.4/×0.8 spread benchmarked to under 2% net saving over Mode A — too weak to
distinguish from noise against the load-based retune. Retuned to ×1.9/×0.55, which cuts peak-period
cost ~30% at the cost of more compressor cycling — see TODO.md §4b's "found via benchmarking" note.)

**Hard safety clamp (§4d):** nothing previously stopped the duty-cycle × tariff multipliers
from stacking arbitrarily wide (up to ~2.4× base band). A real BMS always hard-limits adaptive tuning
to a safe envelope no matter what the cost signal computes — added `[0.5×, 1.5×]` clamp on the final
band.

**Removed: an RH-deviation override that was silently cancelling almost the whole mechanism above
(bug found via audit).** §4d originally also forced the band toward its tight end whenever `zoneRH`
drifted >15 points from `rhTarget`, reasoning that a real adaptive controller shouldn't loosen
temperature control to chase a cheap tariff while humidity is already out of spec. Sound reasoning,
wrong for THIS model: RH here is a passive byproduct of ambient inflow and compressor-driven
dehumidification (see `stepZone()`'s humidity section) — never actively steered toward `rhTarget`,
and for high-target products like potato (`rhTarget` 90%) actual RH settles around 45-47% under
normal operation. That means the override's condition was true essentially 100% of the time,
permanently capping the band at 0.8× base and silently erasing nearly all of the duty/tariff
retuning above it — which is exactly why Two-position and Adaptive looked "almost identical" in
practice despite the tariff-shift logic being real and (on its own) working. Confirmed via a 15-day
headless replay of the real production code: removing the override restored genuine differentiation
system-wide — e.g. tomato's Peak-period kWh dropped ~35% for Adaptive vs Two-position, and several
products (tomato, banana, mango, dairy, produce) flipped from "same or worse" to genuinely cheaper
overall for Adaptive. A real humidity-aware adaptive controller would need RH to actually be actively
regulated first — gating band width on a number the controller can't influence isn't safety, it's
noise that happens to always evaluate true.

**Output:** same shape as Mode A.

**Real-world equivalent:** this is still classical rule-based control, not ML — it matches a BMS
with a manually-programmed adaptive-deadband routine, or a technician who periodically retunes the
deadband from observed trends and the site's known tariff schedule. To connect: same as Mode A, plus
whatever field bus/database the real site's actual duty-cycle log and tariff calendar live in.

## Mode C — Variable-speed (VFD/PI), `cfg.mode === "vfd"`

**Inputs used:** `airTemp[i]`, `target`, `band`, plus the controller's own running integral term
`piIntegral[i]` (pure internal state, carried tick to tick), plus each chamber's own fixed physical
size/capacity (`ZONE_SPECS[i]`) used once to derive its gains, plus (§4d, new) the shift roster
start times and, for Zone A, the booked next truck appointment — both schedule data the sim already
maintains, used only for feedforward, not as privileged foresight a real site wouldn't have.

**Control law:**
```
over = airTemp[i] - target
{Kp, Ki} = vfdGains(i):
    Kp = 1/band                                              // same as before, error-driven
    minutesPerDegreeAtFullTilt = airCapacityKJperK(i)*1000 / compCapacityW[i] / 60
    Ki = 0.004 * (REFERENCE_MIN_PER_DEG=1.5 / minutesPerDegreeAtFullTilt)  // per-chamber commissioning, §4c

feedforward = 0                                              // §4d, new
  + 0.10 if a shift starts within 15 min
  + 0.15 if (zone A only) a booked truck delivery is within 20 min

demandRaw = over*Kp + piIntegral[i] + feedforward
demand = clamp(demandRaw, 0, 1)
satError = demand - demandRaw                                // §4d anti-windup by back-calculation
piIntegral[i] += (over*Ki + satError*KB_ANTIWINDUP=Ki) * dt   // replaces the old blunt ±2 clamp

if 0 < demand < VFD_MIN_SPEED(0.25):             // §4c minimum-speed floor
    demand = (compressor was OFF) ? 0 : (demand < 0.125 ? 0 : VFD_MIN_SPEED)
capacityFraction[i] = slew-limited step toward demand, max 0.06/min
compressorOn[i] = capacityFraction[i] > 0.03
```

**Anti-windup, properly (§4d):** the previous version just clamped `piIntegral` to a hand-picked
±2 — a blunt approximation that lets the integral term keep drifting even while the output is
already saturated at 0 or 1. Replaced with standard back-calculation: the integral only keeps
accumulating error once the output stops being saturated, which is what a real PI/PID
implementation actually does.

**Bug found and fixed during a full-system audit (§4e):** `KB_ANTIWINDUP` was initially a flat `0.5`,
roughly 125x larger than `Ki` (~0.004). Any saturation event — most commonly the temperature spike
right after a routine defrost cycle — drove the integral deeply negative for hours, and if the next
disturbance landed before that unwound, it compounded into permanent drift. A 15-day headless replay
of the actual production code (not a reimplementation) confirmed this concretely: potato held VFD in
its safe band only 1.1% of the time, worse than Two-position/Adaptive's ~60%, backwards from VFD's
whole design intent. Fixing `KB_ANTIWINDUP = Ki` (the standard back-calculation tuning relationship —
same order of magnitude as Ki, not an arbitrary O(1) constant) brought it to 61-68%, now consistently
the *best* of the three across 7 of the 9 product profiles tested — see TODO.md §4e for the full
per-product table.

**Feedforward (§4d):** a real commissioned system doesn't wait for temperature to actually drift
before reacting — it anticipates known disturbances. The sim already knows the shift roster and
Zone A's booked dock appointments in advance (both drive real heat loads: worker body heat, warm
incoming pallets), so a small demand nudge lands shortly *before* the load hits, not only after the
PI term reacts to the resulting error.

**Output:** `capacityFraction[i]` ∈ [0,1] continuous; `compressorOn[i]` derived from it.
Delivered power also folds in `vfdEfficiencyMult(capacityFraction)` — a non-linear part-load
efficiency curve (0.85× rated COP at the speed floor, 1.0× at full speed) — plus (§4d, new)
`VFD_DRIVE_LOSS=0.97`, a small flat inverter/harmonics loss that exists regardless of speed, so
"VFD" isn't implicitly free efficiency at the drive level. The paired variable-speed condenser fan
(§4c/§4d) also now draws its own metered power (`FAN_POWER_FRACTION=0.04` of rated compressor
capacity, scaling down with capacity fraction) — a real, partial offset to VFD's compressor-side
savings that wasn't charged anywhere before.
(Originally 0.72× at the floor; softened after benchmarking showed this alone made VFD look MORE
expensive than Two-position, backwards from the ~15-35% savings VFD retrofits report in the field —
see the corresponding fix on Mode A below and TODO.md §4c.)

**Also relevant — why Two-position, not VFD, carries the bigger energy penalty in this model:**
Mode A's control law (above) now includes a cold-start efficiency penalty: after each OFF→ON
transition, `stepZone()` applies `STARTUP_COP_PENALTY = 0.82` to the delivered COP for
`MIN_RUN_MIN` (3) minutes, representing real refrigerant migration/pressure-equalization losses
after a stop. VFD essentially never incurs this (it rarely fully stops). This — not an invented
VFD bonus — is the actual documented reason smooth modulation beats on/off cycling on energy in the
field, and was the root cause of the benchmarking anomaly described above: without it, on/off's
"when running" power draw was modeled as loss-free, so VFD's own (then-larger) part-load penalty
made it look worse for no compensating reason.

**Condenser head-pressure floating — the bigger effect, added after further benchmarking:** even
with the fix above, VFD's total ₹ was only ~1-3% below Two-position (the cold-start penalty only
covers ~3 of an average ~15-minute cycle, capping its impact). Real VFD retrofits report 15-30%
savings, and that gap turned out to be a genuine missing mechanism, not a tuning knob: a VFD
compressor is standard practice paired with a **variable-speed condenser fan**, which lets head
pressure "float" down when there's less heat to reject — this is the single biggest documented
contributor to VFD savings, bigger than compressor-speed modulation alone, and this simulation had
no condenser-side model at all until now. Added (near `ratedCOP()`):
```
condensing temp = ambient + approach(mode, capacityFraction)
  approach = 12°C fixed                                    // Two-position/Adaptive: fixed-speed fan, no floating available
  approach = 12°C × clamp(capacityFraction, 0.35, 1)        // VFD only: floats down at partial load, floored at 0.35
                                                             // (real systems keep a minimum head pressure for TXV metering)
lift = (ambient + approach) − target
COP multiplier = clamp(42 / lift, 0.6, 1.6)                 // reference lift 42°C = original ratedCOP() design point
effectiveCOP = ratedCOP(target) × COP multiplier
```
`effectiveCOP()` replaces the plain `ratedCOP(target)` call in the power calculation for ALL modes
(so ambient realism — COP dropping on a hot day — now applies universally, which was itself a
pre-existing gap), while only VFD gets the additional floating benefit. Re-benchmarked over 10
sim-days: VFD's kWh dropped ~22% and cost ~15% vs Two-position (₹1728 vs ₹2041) — now in the
real-world-reported range, for a modeled physical reason rather than a tuned multiplier chosen to
hit a target number.

**Real-world equivalent:** a PI/PID loop running on a PLC or the compressor's own onboard
controller, driving a VFD/digital-scroll unit's analog or fieldbus speed reference. To connect: feed
the real `over` (measured − setpoint) into the same PI math (or better, hand this off to the site's
existing PID block and just read/write its output), and write `capacityFraction` out as the real
0–10V/4–20mA/fieldbus speed command instead of a JS number.

## Mode D — AI-based (predictive/optimizing supervisory), `cfg.mode === "ai"`

> **Superseded by "Mode D v2" below.** The control law in this section (90-min horizon, 2-stage plan, flat ₹50/°C·min penalty) is kept for history.

**What "AI-based" honestly means here:** a receding-horizon predictive optimizer (the real technique
behind almost every genuine "AI-based"/"smart" supervisory controller actually deployed in industrial
refrigeration today — see the notes at the bottom of TODO.md). It is explicitly NOT a trained neural
network — this sim has no training data or training infrastructure to make that claim honest, and a
fake "ML" label would be exactly the kind of dishonest description this whole project has tried to
avoid throughout.

**Inputs used** — deliberately no more privileged than what Modes A/B/C already implicitly have:
- `st.airTemp[i]` / `st.zoneTemp[i]` — the zone's own current state, same as the other 3 modes read
  (none of the 4 modes have a separate noisy-telemetry channel for the shadow benchmark instances —
  see the Benchmarking harness note below on why).
- Nameplate `ZONE_SPECS[i]` (rated capacity, UA-equivalent) — NOT the twin's live calibrated
  `estCapMult`/`estUaMult` belief. Mode D forecasts from "nameplate until proven otherwise," the same
  physical-understanding baseline the other 3 modes implicitly have.
- The deterministic SHAPE of the diurnal ambient curve (`ambientForecastAt()` — same formula as
  `ambientNow()`, evaluated at a future time; day-to-day weather variability, §3, is NOT knowable in
  advance, matching real forecast uncertainty) and the known tariff schedule (`tariffPeriodAt()` at
  any future hour).
- Known scheduled events: the shift roster (`SHIFTS`) and, for Zone A, the already-booked
  `state.nextTruckAppt` — a real facility's predictive layer would have this from the same
  dock-appointment/roster system this sim already models, not from magic foresight.

**Control law:**
```
every AI_REPLAN_INTERVAL_MIN (5 min; a real MPC re-solves periodically, not every control tick):
  for each (capFirst, capSecond) in AI_CANDIDATE_LEVELS × AI_CANDIDATE_LEVELS   // 5×5 = 25 trajectories
                                     (AI_CANDIDATE_LEVELS = [0, 0.25, 0.5, 0.75, 1.0]):
    roll forward AI_HORIZON_MIN (90 min, in AI_HORIZON_STEP_MIN=10min steps):
      use capFirst for the first half of the horizon, capSecond for the second half —
        a genuine (if small) TRAJECTORY, not one constant level held the whole time
      forecast ambient/hour via ambientForecastAt(); look up tariffPeriodAt(hour)
      forecast wallW (nameplate UA), shiftW (known roster), and — for Zone A within
        20min of state.nextTruckAppt — the anticipated delivery heat load
      coolingW = nameplate capacity × frostPenalty(current frost) × candidate × (1-latentFraction)
      step airTemp, then zoneTemp (same physics formulas as the real plant)
      accumulate: ₹ cost (forecasted tariff × forecasted power)
                + AI_QUALITY_PENALTY_RS_PER_DEGMIN(50) × max(0, |zoneTemp-target|-band) × step_minutes
  aiPlannedCapacity[i] = capFirst of whichever (capFirst,capSecond) pair had the LOWEST total —
    only the FIRST stage's decision is ever applied; the rest of the plan is thrown away and
    re-optimized from fresh information at the next re-plan, standard receding-horizon practice
  aiNextPlanAt[i] = now + AI_REPLAN_INTERVAL_MIN

every tick: capacityFraction[i] slew-limits toward aiPlannedCapacity[i], max 0.06/min (identical
  hardware limit to VFD mode — see below)
compressorOn[i] = capacityFraction[i] > 0.03
```

**Hardware sharing with Mode C:** Mode D is modeled as driving the SAME modern variable-speed
hardware VFD mode drives (`condenserApproach()`, `vfdEfficiencyMult()`, `VFD_DRIVE_LOSS`, and the
cold-start-penalty exemption all treat `mode==="vfd"` and `mode==="ai"` identically — see
`isContinuousHardware` in `stepZone()`). This is a deliberate modeling choice, not an oversight: real
predictive/MPC supervisory control is deployed almost exclusively on top of variable-speed equipment
in practice, and the interesting difference between Modes C and D is HOW the capacity setpoint gets
chosen (reactive PI vs. lookahead optimization), not what hardware executes it.

**Output:** `capacityFraction[i]` ∈ {0, 0.25, 0.5, 0.75, 1.0} chosen at each re-plan, then
slew-limited toward continuously — same shape as Mode C, driving the plant through the identical
actuator interface.

**Bug found and fixed during verification:** `AI_QUALITY_PENALTY_RS_PER_DEGMIN` was initially set to
`2` — drastically underweighted. A 15-day headless replay showed Mode D correctly minimizing ₹ (it
was cheapest every time) but for several products holding its safe band as little as 0% of the time —
worse than every other mode — because each individual 90-minute re-plan looked locally reasonable
(small predicted deviation) while compounding, across many re-planning cycles, into catastrophic
long-run drift: a classic short-horizon-MPC pitfall, not a bug in the optimization itself. Swept
weights 2/10/30/60/100 and found quality improves dramatically for almost no cost increase well past
2 — settled on `50`.

**Second bug found and fixed** (prompted by a user question: "why is the ₹ gap between VFD and AI
only ₹100-300 even with a predictive optimizer?"): investigated rather than assuming this was fine.
Confirmed the tariff-shifting mechanism WAS genuinely working (AI's Peak-period kWh was measurably
lower than VFD's), but lengthening the forecast horizon (90→180→240→300 min) did NOT increase AI's
savings advantage — which it should have, if horizon length were the binding constraint. Root cause:
the original version evaluated each candidate as one CONSTANT capacity level held over the entire
horizon — structurally unable to express "run harder now, ease off later," exactly the shape needed
to properly bank thermal mass ahead of a tariff transition. Fixed by upgrading to the genuine 2-stage
piecewise trajectory described above. Re-verified across all 9 products: quality improved
substantially (e.g. potato in-band 64.8%→78.1%, tomato 40.3%→56.4%) at similar or lower cost — Mode D
is now cheapest in 8 of 9 products (banana trades a small cost increase for a large quality gain — a
legitimate choice for a chilling-sensitive product) and best-in-band in 8 of 9 (dairy is essentially
tied with VFD). The remaining ~₹100-300 gap between VFD and AI (vs. the much larger ~15-20% gap
between relay-based modes and VFD/AI) is now a **verified, not assumed, finding**: the big savings
come from the HARDWARE difference (continuous modulation vs. relay cycling); the further value of
*prediction* on top of already-good continuous hardware is inherently capped by how much thermal
"storage" the product's own allowed temperature band provides — matching real-world reports that MPC's
incremental savings over well-tuned VFD/PID refrigeration are typically single-digit percent, not the
same order of magnitude as the on/off-to-VFD jump.

---

## Mode D v2 — 12-hour economic MPC with demand charges

**Why it was redesigned.** A 135-day headless replay of v1 showed AI used *exactly* the same
Peak-period kWh under a flat tariff as under a time-of-day tariff: it was not shifting load at
all, and its whole edge over VFD/PID was ~0.3% of the bill. Causes: a 90-minute horizon against
product time constants of 9-55 h; no value for cold stored at the end of the horizon; a flat
₹50/°C·min penalty that dwarfed the ₹1-2 price signal per step; and no knowledge of demand
charges or restart losses.

**Inputs (all known in advance to a real supervisory layer):** product temperature, frost and
RH per zone; nameplate UA/capacity (not the twin's estimates); the smooth seasonal ambient curve
(not the day's weather noise); the tariff bands and the state's demand charge; the shift roster;
each zone's defrost schedule; Zone A's booked delivery (door heat + warm-stock blend). The
put-away after a delivery is deliberately *not* modeled — its timing is crew-dependent, and a
version that knew it exactly learned to game it (see "Put-away fix" below).

**Control law:**
```
every AI_REPLAN_INTERVAL_MIN (5 min), for zone 0, 1, 2 in turn (Gauss-Seidel):
  othersKW[k] = the other zones' latest planned draw, shifted to now
  plan = 7 stages over 12 h (30m, 30m, 1h, 2h, 2h, 3h, 3h), one level each
         from AI_LEVELS = [0, .25, .4, .55, .7, .85, 1]
  start from the previous plan (shifted) and every constant level; then 2 sweeps of
  coordinate descent over the stages (~90 rollouts per zone)
  rollout (15-min steps): air solved quasi-steady, product integrated with respiration,
    delivery blend, defrost minutes, frost growth; power from the same COP/VFD/fan chain
    the plant uses
  cost = Σ energy kWh × ToD rate
       + demand rate × max(0, horizon peak kVA − max(month MD so far, billing floor))
       + demand rate × (2−1) × max(0, peak kVA − contract demand)
       + restart cost (cold-start loss) for every 0 → on transition
       + penalty × °C·min outside the band shrunk by 15% margin
       + penalty × °C·min of supply air more than 3 °C below the band
       + (end − start product heat content) priced at the next 12 h's average tariff
  apply only stage 1; hardware path identical to VFD (rackContinuousStep — VFD-lead rack since v3)
```
Penalty = `AI_QUALITY_PENALTY_FULLPOWER_MIN` (60) minutes of that zone at full rated power per
°C·min. Swept 15/30/60/120: results moved by <0.5% of the bill and not at all in time-in-band.

**Benchmark at the time of v2 (superseded by the v3 table below — v3 added staged racks, fan heat and put-away heat transfer, which changed every number):**

| Scenario (135 days, commercial store) | Rule-based | PID (VFD) | AI | AI vs rule | AI vs PID | Safe range: rule / PID / AI |
|---|---|---|---|---|---|---|
| Leafy produce, summer, national | ₹20.6 L | ₹18.1 L | ₹18.0 L | 12.5% | 0.5% | 84.9 / 93.5 / 94.9% |
| Leafy produce, winter, national | ₹13.0 L | ₹12.7 L | ₹11.7 L | 9.8% | 7.8% | 97.5 / 100 / 100% |
| Leafy produce, monsoon, Tamil Nadu | ₹22.4 L | ₹20.3 L | ₹20.3 L | 9.3% | 0.0% | 89.0 / 93.6 / 99.7% |
| Potato, summer, Maharashtra | ₹14.2 L | ₹12.7 L | ₹12.0 L | 15.7% | 5.3% | 92.4 / 94.8 / 98.1% |
| Potato, summer, Uttar Pradesh | ₹10.5 L | ₹9.4 L | ₹8.9 L | 15.3% | 5.5% | 92.4 / 94.8 / 98.1% |
| Potato, winter, national | ₹6.1 L | ₹6.1 L | ₹5.7 L | 6.8% | 6.9% | 99.2 / 99.4 / 99.9% |
| Frozen meat, summer, national | ₹16.1 L | ₹15.4 L | ₹14.3 L | 11.3% | 7.6% | 97.9 / 97.9 / 97.9% |
| Frozen meat, winter, national | ₹8.7 L | ₹8.2 L | ₹7.4 L | 14.7% | 9.5% | 98.0 / 98.0 / 98.0% |
| Tomato, summer, national | ₹16.9 L | ₹15.3 L | ₹15.5 L | 8.5% | **−0.9%** | 87.4 / 91.7 / 90.6% |
| Tomato, winter, national | ₹12.1 L | ₹11.8 L | ₹11.3 L | 6.7% | 4.5% | 76.8 / 100 / 100% |
| Banana, summer, Gujarat | ₹17.8 L | ₹17.0 L | ₹16.7 L | 6.6% | 2.1% | 78.7 / 68.5 / 90.2% |

Bill = energy + maximum-demand charges. "Safe range" = % of time every chamber's product core is inside its band.

AI now beats rule-based everywhere (6.6-15.7%), beats PID in 10 of 11 scenarios, and has the best
or tied-best time in safe range in 10 of 11. The exception is tomato in summer (fast τ=9 h, short
delivery cadence), where PID is 0.9% cheaper and holds band slightly better. AI's lead over PID is
largest where there is storable cold and a real price spread (potato, frozen meat, winter).

## Changes shipped alongside Mode D v2 (affect every mode)

- **Maximum-demand charge** (`stepDemandMeter`): per-run meter integrating 15/30-min blocks,
  kVA at PF 0.92 for every controller, monthly MD billed at max(MD, floor × contract demand),
  excess over contract demand billed 2x, 30-sim-day billing months, running month pro-rated.
  Each benchmark shadow has its own meter. Shown in the header, the bill breakdown, the
  comparison modal and the scoreboard.
- **14 state tariffs** (`TARIFF_REGIONS`): 8 added (Karnataka, Tamil Nadu, Telangana, Andhra
  Pradesh, Haryana, MP, Rajasthan, Bihar) and every region now carries `demandRsPerKVA`,
  `mdFloorFrac`, `mdBlockMin`. Representative approximations — verify against the current
  tariff order before using any number as a real rate.
- **Commercial facility** (`FACILITY_PRESETS.commercial`, default): ~2,650 t across 3 chambers;
  plant auto-sized per product to 1.5x the summer design load (`buildZoneSpecs`), contract
  demand from the design-day draw + 10% (`computeContractKVA`). The original rooms remain as
  "Small demo rooms". **Heat of respiration** (`respWPerT`) added — negligible at 35 t, a major
  load at 2,000 t.
- **Restart losses for VFD/AI** (now per unit in `startUnit`, v3): the cold-start COP penalty used to
  be waived for continuous hardware; replay showed VFD/AI restarting ~170x/day in winter for
  free. Now every restart pays it, plus a soft-start energy charge, under the same 3/2-min
  run/rest timers as relay modes. This is why PID now ties rule-based for potato in winter.
- **Put-away fix**: put-away blended Zone A's product toward the *air* temperature and also fired
  on outbound-truck door cycles. Now it only follows a delivery and returns the zone toward
  the pre-delivery stock temperature; it is also mirrored into the benchmark shadows (it used
  to happen in the live run only).
- **Runs start at setpoint** instead of a fixed 8/8/−5 °C, so the first days don't count a
  commissioning transient against every controller.
- **Twin estimator** now includes air↔product coupling (dominant at commercial scale), and the
  3-hour forecast integrates in 1-minute substeps (5-minute explicit steps went unstable with
  commercial-size compressors).

## Staged racks & realism pass (v3) — current

**Staged compressor racks** (`buildRack`, `rackRelayStep`, `rackContinuousStep`). Commercial
chambers split their plant into ~100 kW units (2-4 per chamber; demo rooms keep 1):
- *Relay rack* (Two-position, Adaptive): equal fixed-speed units on a step thermostat. Stage k+1
  cuts in at `airSP + band·((k+1)/n − ½)` and out `band·(½ + ½/n)` below that — a ladder centred on
  the setpoint; n=1 is the ordinary thermostat `airSP ± ½·band`. **Bug fixed:** the old single-unit
  thermostat cut in at target + the whole product band (the product's quality limit) and out at
  −0.3·band, so the product sat on its upper edge — demo Zone A was in range 23% of summer under
  Two-position vs 89% under VFD; now 93%. An uncentred ladder also had load-dependent droop (two of
  three stages held bulk tomato ~0.55·band warm). Successive starts ≥ `INTERSTAGE_DELAY_MIN` apart.
- *Air setpoint trim* (`airSetpointTrim`, relay and PID): thermostats/PI act on air but quality is
  judged on the product core, which runs warmer by respiration ÷ air↔product coupling; the air
  setpoint is commissioned that much below target (≤ ½·band). PID potato in-range 92.7% → 96%.
- *Adaptive band* now applies its tariff strategy as a setpoint schedule (−0.3·band in Off-Peak/
  Solar, +0.35·band in Peak, bounded so the thermostat cycle stays ≥0.2·band inside the product
  band — unbounded, winter pre-cooling chilled tomato to 79% in range) rather than band width, which stopped moving the mean once the
  thermostat was centred; duty-based differential tightening/relaxing is unchanged.
- *VFD-lead rack* (PID, AI): one variable-speed lead sized `VFD_LEAD_SIZE_RATIO` (1.4) × a lag,
  plus fixed-speed lags. A lag starts after the lead has been ≥97% for `STAGE_UP_DELAY_MIN`, stops
  after the lead has wanted < its floor for `STAGE_DOWN_DELAY_MIN`. The oversized lead makes its
  range overlap the fixed steps (with equal sizes, demand between 1.0 and 1.25 units hunted).
- Lead-lag rotation by run-hours; per-unit anti-short-cycle: 5 min run, 3 min rest, ≥10 min
  start-to-start (≤6 starts/h, manufacturer guidance for semi-hermetic compressors).
- Starters: VFD soft start; fixed units > 11 kW star-delta (~1.9× rated), smaller DOL (~5.8×).
- Power summed per unit: VFD lead pays part-load + drive losses, lags run at full-load COP, any
  unit within 3 min of its start pays the cold-start penalty. Condenser fans stage with the
  compressors on relay racks and float on VFD racks.

**PI commissioning** (`vfdGains`): lambda/IMC tuning from each chamber's identified first-order air
response (gain from capacity vs UA + air↔product coupling, τ from air mass), θ = 4 min sensor lag,
λ = 10 min. Replaces Kp = 1/band, which on a 650 kW receiving plant asked for ~160 kW per 0.5 °C
and limit-cycled the lead every 6 minutes. λ 4/6/10 changed time-in-band by <0.5 point; 10 gives
the fewest starts.

**Other realism fixes in this pass**
- Evaporator fans (~2% of capacity, `evaporatorFanKW`): continuous on fixed-speed systems, EC fans
  ∝ speed³ on VFD racks; their power is billed and their heat lands in the room (twin, forecast
  and planner include it). Standby draw cut to controls + crankcase heaters (0.3%).
- Put-away pallets now carry their remaining heat into the destination chamber (they used to
  vanish), and the AI forecast models put-away at the WMS-scheduled time for both chambers.
- Frost growth and dehumidification scale with coil load, not just "any compressor on".
- Wear runtime is equivalent full-load hours; contactor wear is per unit (VFD-started units have
  none).
- Twin forecast runs "at current duty" (last-hour capacity share) instead of a single
  full-capacity thermostat, which was wrong — and unstable — for staged 650 kW plants.
- Contract demand is user-settable; the comparison shows each controller's right-sized contract and
  its annual demand-charge saving. Advisory panel reads the MPC's actual plan (`aiPlanAdvisories`).

**Benchmark (135 days, commercial store unless noted, seed 12345; bill = energy + demand charges).**
Final state of v3 — racks, fan heat, put-away heat, centred thermostat, setpoint trim:

| Scenario | Rule-based | PID | AI | AI vs rule | AI vs PID | Safe range rule / PID / AI | Starts/day, whole plant (rule / PID / AI) |
|---|---|---|---|---|---|---|---|
| Potato, summer, Maharashtra | ₹17.50 L | ₹14.49 L | ₹13.67 L | 21.9% | 5.7% | 98.2 / 96.5 / 97.2% | 123 / 32 / 29 |
| Potato, summer, Uttar Pradesh | ₹13.20 L | ₹10.74 L | ₹10.11 L | 23.4% | 5.9% | 98.2 / 96.5 / 97.3% | 123 / 32 / 29 |
| Potato, summer, national | ₹13.09 L | ₹10.72 L | ₹10.19 L | 22.2% | 5.0% | 98.2 / 96.5 / 97.4% | 123 / 32 / 31 |
| Potato, winter, national | ₹8.29 L | ₹6.55 L | ₹6.35 L | 23.5% | 3.0% | 100 / 99.6 / 100% | 128 / 47 / 52 |
| Onion, winter, Madhya Pradesh | ₹7.70 L | ₹5.96 L | ₹5.53 L | 28.2% | 7.3% | 100 / 99.6 / 100% | 64 / 46 / 56 |
| Frozen meat, summer, national | ₹18.80 L | ₹16.28 L | ₹15.40 L | 18.1% | 5.4% | 98 / 98 / 98% | 145 / 11 / 18 |
| Frozen meat, winter, national | ₹11.66 L | ₹9.12 L | ₹8.06 L | 30.9% | 11.6% | 98.4 / 98 / 98% | 156 / 21 / 25 |
| Tomato, summer, national | ₹23.12 L | ₹19.45 L | ₹18.58 L | 19.6% | 4.5% | 91.4 / 91.4 / 91.5% | 163 / 31 / 35 |
| Tomato, winter, national | ₹16.24 L | ₹12.17 L | ₹12.02 L | 26.0% | 1.3% | 100 / 100 / 100% | 81 / 52 / 47 |
| Leafy produce, winter, national | ₹15.57 L | ₹13.31 L | ₹13.45 L | 13.6% | −1.0% | 100 / 100 / 100% | 107 / 29 / 44 |
| Leafy produce, summer, national | ₹24.37 L | ₹20.66 L | ₹21.20 L | 13.0% | −2.6% | 94.5 / 93.7 / 96.2% | 174 / 22 / 43 |
| Leafy produce, monsoon, Tamil Nadu | ₹26.96 L | ₹22.74 L | ₹23.60 L | 12.5% | −3.8% | 97.7 / 93.6 / 99.7% | 168 / 22 / 36 |
| Mango, summer, Andhra Pradesh | ₹24.03 L | ₹20.54 L | ₹20.61 L | 14.2% | −0.3% | 95.9 / 95.9 / 96.0% | 182 / 23 / 60 |
| Banana, summer, Gujarat | ₹23.16 L | ₹20.15 L | ₹20.26 L | 12.5% | −0.5% | 91.6 / 87.5 / 91.7% | 278 / 25 / 48 |
| Leafy produce, winter (demo rooms) | ₹0.84 L | ₹0.67 L | ₹0.64 L | 24.6% | 4.8% | 100 / 100 / 100% | 136 / 104 / 35 |
| Leafy produce, summer (demo rooms) | ₹1.48 L | ₹1.16 L | ₹1.17 L | 21.2% | −0.3% | 92.9 / 93.5 / 90.8% | 181 / 11 / 17 |

**Reading it.** AI beats rule-based everywhere (12.5-31%). Against a properly commissioned PID
(lambda-tuned, setpoint-trimmed, on the same VFD-lead rack) AI is cheaper wherever there is
storable cold and a real price spread — potato, onion, meat, tomato: 1.3-11.6%. On fast-respiring,
short-τ produce (leafy greens, mango, banana) it is within 0.3-3.8% of PID's bill and spends that
on a higher or equal time in range (e.g. Tamil Nadu monsoon: 99.7% vs 93.6%). That is the honest
size of the predictive layer's value on top of good VFD hardware — the hardware itself is worth
12-25% over relay control. Adaptive band (tariff setpoint schedule, bounded inside the band):
0.9-2.5% under Two-position at the same time in range.

## Mode E — AI (self-learning MPC), `cfg.mode === "ai"` (and Mode D renamed `"mpc"`)

**Naming.** The predictive controller previously labelled "AI-based" is now **Predictive optimizer
(MPC)**, key `"mpc"`, unchanged: a physics model plus an optimizer, with no learning. Key `"ai"` is
the new mode: the *same* planner plus an online learner. Both run every tick, so the MPC row is the
learning-off baseline for the AI row.

**Something to learn: as-built deviations** (`asBuilt`, drawn from the run seed). Every controller,
the twin and the plant sizing use nameplate numbers; the simulated building differs, like a real
one: insulation UA 110-135% of design, compressor capacity 85-97% of catalogue, respiration 80-130%
of the textbook figure. Applied to the plant physics only, identical for all controllers.

**What the learner does** (`freshLearner`, `learnerObserve`, from BMS-available measurements only):
1. **RLS on the measured energy balance**, 15-min averaged windows, ~3-day forgetting:
   `(C_air·ΔT_air + C_prod·ΔT_prod)/Δt − nameplate model = θ0 + θ1·wall heat − θ2·cooling`.
2. **Hour-of-day heat table** on what RLS leaves unexplained (recurring loads).
3. **Delivery share** replaced in Zone A (from the measured product-temperature jump) and the
   **put-away delay** after the door closes.
4. **Weather offset** of today's ambient from the seasonal curve, faded over the horizon.
5. **Per-chamber safety margin** tuned on outcome: widens when that chamber's product is found
   outside its band (except Zone A while a delivery pulls down — unavoidable), relaxes slowly.

**Estimator safeguards (found by 135-day replays — 30-day runs looked fine):** the first version
diverged after ~a month (model error exploding, AI time-in-range falling to 75-90%). Fixed with the
standard industrial safeguards: windows containing a delivery or put-away are discarded (an instant
stock swap is a mass exchange, not a heat flow); outlier windows (>5σ of recent error) are skipped;
the RLS covariance trace is capped (forgetting with weak excitation otherwise winds it up); θ is
clamped to physical ranges. After the fix the learned error stays at 0.2-0.5 kW for 135 days.

The planner (`aiPlanZone`) then uses learned UA, heat bias, hour table, delivery share, put-away
delay, weather offset and margins. **Learned capacity (θ2) is a diagnostic only**: from closed-loop
operating data it is biased 3-4% low (the controller raises capacity exactly when unexplained heat
arrives, so "weaker compressor" and "more heat" can't be cleanly separated — classic closed-loop
identification bias), and planning with it over-cooled and cost 1-2% more in testing.

**Evidence the learning works** (AI learning panel, `learningLog`): RMS model error in explaining
measured heat flow fell from ~5-7 kW (nameplate) to ~0.3-0.5 kW (learned) within ~4 days;
learned extra heat tracks the hidden truth (e.g. Bulk zone −11 kW learned vs −13 kW true), capacity
estimates within 1-2%. Insulation vs respiration errors are confounded in operating data, so the
panel reports their combined effect, which is what the planner needs.

**What it buys — honestly:** see the table below. MPC re-plans every 5 minutes from *measured*
temperatures, so feedback already corrects most of a wrong model; learning makes the model right
but turns into only ±1% on the bill in this simulation. Where learning earns more in practice: a
building that drifts over months (fouling, seal failure), sites without good spec data, and
self-diagnosis — the same estimates flag degradation (the twin now baselines its alarms at
commissioning for the same reason).

**Twin baseline fix (all modes).** With as-built deviations the twin's EKF correctly sees a ~5-15%
capacity shortfall from day one, which raised "service due" immediately. Alarms are now relative to
a commissioning baseline taken at 40 equivalent full-load hours, as real monitoring systems do.

## Realism audit (v4) — false assumptions found and fixed

A deliberate pass over every physical and operational assumption, asking "is this how a real
Indian cold store behaves?". Found and fixed (all controllers affected alike):

| # | Was | Now |
|---|---|---|
| 1 | Every room ran an 18-min **electric** defrost every 6-8 h — incl. 12-14 °C tomato/banana rooms whose coils never frost | Method by coil temperature (`defrostPolicy`): no defrost if the coil stays above 0 °C; **off-cycle** (compressor off, fans on, no heater, 30 min) for 0-8 °C rooms; electric only below 0 °C. Frost forms only when the coil is below 0 °C |
| 2 | COP = ad-hoc linear function of target × capped lift multiplier (frozen-room COP ~2.3, no winter floor) | Lift-based cycle model (`cycleCOP`): η·T_evap/(T_cond−T_evap), η=0.55, coil TD shrinking at part load, 25 °C head-pressure floor. 4 °C room/34 °C day → 3.0; −18 °C → 1.9 |
| 3 | **Evaporator delivered full rating however cold the air got** — an oversized unit on its min-run timer drove a 4 °C potato room's air to −0.6 °C (product below its band); the planner once forecast −9 °C air | Coil output ∝ (air − coil temperature) (`coilCapacityFactor`), 0 at the coil temperature, up to 1.5× on pull-down; compressor power follows. Planner solves the air balance with it |
| 4 | One fixed season for a whole 135-day run | Calendar option (`seasonAt`): monthly climate normals interpolated day by day from a start month — default is now "Calendar from 1 Apr" (summer into monsoon) |
| 5 | Envelope saw only air temperature | Roof **sol-air** gain (`solAirExcessC`): clear-sky irradiance by month, cut on overcast days, α 0.45 → up to ~25 °C extra on the roof |
| 6 | Produce rooms sat at 45-65 % RH drifting to outdoor RH; latent load from outdoor RH | Transpiration pushes RH toward saturation, door air toward outdoor RH, coil condenses ∝ load → produce rooms 80-90 %; latent share from **room** RH (`latentFractionFor`) |
| 7 | Bulk/Deep-freeze perfectly sealed; no lighting | Their doors open (strip curtain, 85 %) whenever the forklift works inside; LED lighting 4 W/m² while occupied — billed and heating the room |
| 8 | Controllers acted on exact ground truth | Thermostat/PID read the lagged return-air sensor with calibration offset (±0.3 °C) and noise; MPC/AI a product probe (±0.2 °C offset) through a 15-min filter. Noise/offsets shared by all controllers |

Each fix exposed or fixed controller behaviour too: (3) restored Two-position potato to 98.7 %
in range; (8) without the probe filter, plans flip-flopped (~5× more starts) — real supervisory
controllers filter measurements, so the filter is part of the realism.

**Still knowingly simplified** (documented, not hidden): all chambers hold one product at one
setpoint (per-chamber setpoints parked by earlier instruction); each chamber well mixed (no
stratification); air-cooled condensers (many large potato stores use ammonia with evaporative
condensers — better summer COP); no grid outages / DG-set backup (common on rural Indian feeders);
bill excludes electricity duty, fixed charges and fuel-adjustment; tariff and demand figures are
representative approximations, not a live tariff order; climate normals are Indo-Gangetic-plain
style for every state.

## Current benchmark (final code: realism audit + per-chamber products + location climate)

135 days, commercial store unless noted, seed 12345, hidden as-built deviations, controllers on
real sensors. Bill = energy + maximum-demand charges. Supersedes all earlier tables in this file.

| Scenario | Rule-based | PID | MPC | AI | AI vs rule | AI vs PID | Safe range rule / PID / MPC / AI |
|---|---|---|---|---|---|---|---|
| Frozen meat, winter, National | ₹13.01 L | ₹10.72 L | ₹10.03 L | ₹9.89 L | 24.0% | +7.8% | 98.5 / 97.9 / 97.9 / 97.6% |
| Frozen meat, summer, National | ₹19.18 L | ₹15.88 L | ₹15.16 L | ₹14.92 L | 22.2% | +6.1% | 98.3 / 97.9 / 97.9 / 97.5% |
| Onion, winter, MP (Indore) | ₹9.96 L | ₹7.58 L | ₹6.93 L | ₹6.95 L | 30.2% | +8.2% | 99.6 / 99.4 / 99.2 / 98.9% |
| Potato, winter, Punjab (Ludhiana) | ₹9.11 L | ₹6.91 L | ₹6.57 L | ₹6.56 L | 28.0% | +5.1% | 100 / 99.9 / 99.6 / 99.3% |
| Potato, calendar from Mar, UP (Lucknow) | ₹15.75 L | ₹11.82 L | ₹11.54 L | ₹11.47 L | 27.2% | +2.9% | 98.6 / 98.3 / 97.9 / 97.9% |
| Potato, summer, UP (Lucknow) | ₹16.37 L | ₹12.37 L | ₹12.11 L | ₹12.09 L | 26.2% | +2.3% | 98.3 / 97.5 / 96.6 / 97.1% |
| Tomato, summer, National | ₹28.52 L | ₹20.45 L | ₹19.35 L | ₹19.37 L | 32.1% | +5.3% | 92.8 / 91.6 / 92.8 / 93% |
| Leafy produce, summer, National | ₹29.06 L | ₹23.93 L | ₹24.98 L | ₹24.55 L | 15.5% | -2.6% | 95.6 / 93.7 / 96.2 / 96% |
| Leafy produce, winter, National (demo rooms) | ₹0.98 L | ₹0.74 L | ₹0.72 L | ₹0.72 L | 27.0% | +3.1% | 100 / 99.8 / 99.5 / 99.6% |
| Potato + Frozen meat in C, calendar from Apr, UP (Lucknow) | ₹17.33 L | ₹13.17 L | ₹13.00 L | ₹12.90 L | 25.6% | +2.1% | 98.4 / 97.8 / 96.3 / 94.7% |
| Leafy produce, summer, National (demo rooms) | ₹1.76 L | ₹1.36 L | ₹1.42 L | ₹1.41 L | 20.0% | -3.8% | 93.8 / 93.7 / 90.6 / 93.9% |
| Leafy produce + Frozen meat in C, calendar from Apr, Karnataka (Bengaluru) | ₹24.72 L | ₹19.67 L | ₹20.34 L | ₹20.13 L | 18.6% | -2.3% | 99 / 96.3 / 98.6 / 97.3% |
| Potato + Onion in C, calendar from Mar, WB (Kolkata) | ₹19.98 L | ₹15.16 L | ₹14.73 L | ₹14.57 L | 27.1% | +3.9% | 92.7 / 88.4 / 91.9 / 91.8% |
| Mango, summer, AP (Vijayawada) | ₹32.21 L | ₹25.42 L | ₹25.88 L | ₹25.75 L | 20.1% | -1.3% | 95.8 / 95.8 / 95.8 / 95.8% |
| Leafy produce, summer, Tamil Nadu (Chennai) | ₹36.54 L | ₹31.04 L | ₹32.58 L | ₹31.98 L | 12.5% | -3.0% | 94.5 / 93.6 / 94.9 / 94.9% |
| Banana, summer, Gujarat (Ahmedabad) | ₹29.29 L | ₹22.78 L | ₹22.83 L | ₹22.83 L | 22.0% | -0.2% | 92 / 91.5 / 92.2 / 92.3% |

**Reading it.** AI beats rule-based everywhere (12.5-32 %) and beats PID in 10 of 16 scenarios
(+2 to +8 %) — storable cold and steadier products (potato, onion, frozen meat, tomato) and the
mixed stores. On fast-respiring produce in summer (leafy greens, mango, banana, and leafy greens in
the Bengaluru mixed store) a well-commissioned PID on the same VFD-lead racks is 0.2-3.8 % cheaper:
those rooms can bank little cold and steady part-load running is already near-optimal; AI spends
the difference on equal or better time in range. The fixed-model MPC is never better than the AI by
more than 0.3 % and is up to ~2 % worse where the building differs from its spec sheet. Variable-
speed hardware (PID) is worth 15-28 % over relay control on its own; prediction adds a few % on top where
there is something to shift, and learning adds up to ~2 % on top of prediction.

## Configurability & controller fixes (v5)

**Per-chamber products** (`cfg.zoneProducts`, `zp(i)`): each chamber holds its own commodity at its
own target/band; the plant, defrost method, setpoint trim, PI tuning, planner, learner and scoring
are all per chamber. Receiving (A) pre-cools what B or C holds and its trucks are put away there.
**Direct delivery streams** (`checkDirectDeliveries`): a chamber whose product isn't received
through A (e.g. frozen meat in C) gets its own reefer deliveries straight into the chamber on its
own cadence — staging frozen stock in a +4 °C room would warm it. The planner forecasts them.

**Location → climate** (`CLIMATE_BY_REGION`): each state uses monthly normals for a representative
cold-storage city (Lucknow, Ludhiana, Hisar, Jaipur, Kolkata, Patna, Indore, Pune, Ahmedabad,
Bengaluru, Chennai, Hyderabad, Vijayawada); fixed seasons are that city's own months averaged, and
the plant's design temperature is the hottest month's afternoon + 2 °C (`designAmbientC`).

**Controller fixes found by the 135-day replays after the realism audit**
- *Pull-down protocol*: excursions while delivered stock cools back into band are priced at the
  shelf life they actually cost (`spoilageRsPerDegMin`, from product value ₹/kg, Q10 and shelf
  life), not at the band-violation weight (4-30× higher), which ran the receiving room flat out at
  poor COP after every delivery.
- *Move suppression* (`AI_MOVE_SUPPRESSION_H`): standard industrial-MPC cost on large step
  changes. The planner had run racks bang-bang (0 %/100 %), losing ~23 % COP at full load plus ramp
  and restart losses; ~30 % fewer starts with it.
- *Margin tuning* skips only while delivered stock is actually still above band (it skipped for
  1.5 τ = 82 h on frozen meat, leaving the margin untuned).
- *Learner validation*: the planner uses learned values only while they explain the data better
  than nameplate; if clearly worse, that chamber's learner restarts (a rare late divergence used to
  persist).

**Honest finding**: the fixed-model MPC can lose to a well-tuned PID on fast-respiring produce when
the building differs from its spec sheet (e.g. produce respiring 16 % less than the textbook figure
→ MPC over-cools). The self-learning AI corrects exactly that and matches or beats PID there.

## Benchmarking harness (§6) — what it measures and how to read it

**All 4 modes run genuinely in parallel, always, regardless of what's on screen.** The visualized
`state` (full twin/telemetry/events/UI) is driven by whichever mode is picked in the header — but
independently of that, four lightweight physical "shadow" instances (`shadowStates.rule`,
`.adaptive`, `.vfd`, `.ai`, see `freshShadowState()`/`SHADOW_KEYS`) step every tick regardless of the
picker, each running its OWN controller against the SAME shared sim clock, ambient/tariff schedule,
and the SAME mirrored dock-door/truck-delivery/stock-turnover events as the live run (see
`checkTruckSchedule`/`maybeRotateStock`'s shadow-mirroring loops and `tick()`'s shadow-stepping
block). A shadow deliberately skips the twin estimator, telemetry noise, and event log — it needs
ground-truth physics + control + energy bookkeeping for a fair benchmark, not a duplicated sensing
simulation per mode (this is also why Mode D's own forecast, above, uses nameplate specs rather than
a live twin belief — no shadow has a twin running to belong to). Selecting a mode in the header only
changes which one is animated/visualized; it does not pause or reset the other three, and switching
back later finds them still running with uninterrupted history.

**Per mode, tracked continuously in `shadowStates[mode].modeMetrics[mode]`:**
| Metric | Field | Meaning |
|---|---|---|
| Time driven | `minutesTracked` | sim-minutes this mode has been running (grows in lockstep across all 3 — they never stop) |
| Cost by tariff period | `costByPeriod[periodKey].{kwh,cost}` | ₹ and kWh split across offpeak/normal/solar/peak |
| Time-in-band | `minutesInBand` | minutes ALL 3 zones were within `target ± band` of product core temp |
| Excursion severity | `degMinOutside` | Σ (°C beyond band × minutes) — deviation size × duration, across all zones |
| Compressor cycling | `cyclesStarted` | OFF→ON transitions, summed across zones — a wear proxy |
| Shelf-life consumed | `spoilStart` / `spoilLast` | worst-zone spoilage index (0–100) at first vs. most recent tick this mode has run |

**Access points:**
- UI: the "Controller comparison" header badge/modal — live table, one row per mode, all four
  updating simultaneously. Every column header and the mode-name cell has a hover explanation
  (same `data-tip`/`TIPS` mechanism used everywhere else on the dashboard, keys prefixed `cmp:`).
- Programmatic: `window.ColdStorageTwin.getModeMetrics()` — returns a deep-cloned snapshot of all
  four shadows' ledgers, safe to poll from an external script/notebook for offline analysis.

**How to run a fair benchmark:** there is nothing to set up — all four have been accumulating
since the sim started (or since the last "Reset run"). Just open "Controller comparison" and read
the rows; `minutesTracked` is identical across rows at any given moment, so raw totals are already
comparable without normalizing — though ₹/kWh, %-in-band, and cycles/hour are still reported
because they're the more meaningful units for judging a controller's behavior, not just its scale.

**Why the rows are safe to compare at all (fixed bug):** every stochastic disturbance the four
parallel runs are exposed to is shared — ambient, tariff clock, dock-door state, truck/turnover
events, AND the equipment-wear random walk (`gaussianNoise(0.0006)` inside `trueCapMult`, drawn once
per zone per tick in `tick()` as `sharedWearNoise` and passed into every `stepZone()` call). This was
found as a bug: that noise term was originally drawn independently per shadow; its random-walk std
(~2.3%/day, ~7%/10-days) was larger than the real ~1-3% Two-position-vs-Adaptive signal, so which
mode looked cheaper could flip purely by chance — confirmed with a 200-trial Monte Carlo test (44%
flip rate with independent noise, 0% after sharing it). If you ever add a NEW stochastic element to
`stepZone()`/`stepCompressor()`, it needs to be either shared the same way or consciously left
independent (documented why) — an unshared random term is a live risk of silently corrupting this
comparison again.

**Reproducibility (§2):** every run is seeded (`rng()`, a `mulberry32` PRNG — see the "Seed" field in
Controls). Re-running the exact same seed produces a bit-identical run across all four modes,
including the AI-based one's own re-planning decisions — verified this specifically, since Mode D's
planning loop is the newest and most complex piece of stochastic-adjacent logic in the sim.

---

## §4d gap-analysis pass — deliberately deferred items

Not every real-world gap found in this pass was implemented — some carry real architectural risk or
low value for the effort. Tracked in `TODO.md` §4d rather than silently dropped:
- ~~Multiple staged compressors per zone (lead-lag)~~ — implemented in v3 (see "Staged racks & realism pass").
- Locked-rotor inrush as a real current-vs-time profile (rather than a flat one-time kWh charge).
- Adaptive-band memory across days (day-of-week/hour learned patterns).
- Contactor/relay electrical wear as its own failure mode, distinct from the compressor capacity
  wear §4d already added from cycle count.
