# Controller I/O contracts — exact inputs and outputs, per mode

This is the reference for "what would it actually take to swap this for a real building."
Every controller mode is a pure function of a small, explicit set of inputs, producing a small,
explicit set of outputs. If you're integrating this against real hardware later, this file is the
seam: replace the *inputs* with a real feed and the *outputs* with a real actuator command, and the
control logic itself does not need to change.

Code locations: `stepCompressor(i, dt)` in `simulation/cold_storage_simulation.html` (~line 1822
at time of writing — search for `function stepCompressor`).

---

## Shared inputs (all modes read these)

| Input | Simulated source today | Real-world source it stands in for |
|---|---|---|
| `airTemp[i]` (chamber air temperature) | Physics engine's AIR node | A wall/duct-mounted RTD or thermistor, via the chamber's BMS/PLC analog input |
| `cfg.product.target` | Selected product profile | The setpoint configured on the thermostat/BMS for the active SKU |
| `cfg.product.band` | Selected product profile | The configured hysteresis/deadband width |
| `dt` (tick size, minutes) | Simulation loop | Real controller scan/loop interval (typically sub-second to a few seconds for a PLC) |
| `state.tMin` → hour-of-day → `tariffPeriodAt(hour)` | Hardcoded 4-band ToU schedule (`TARIFF_BANDS`) | The real DISCOM's actual industrial/commercial ToU tariff schedule (state-specific — see §3 in TODO.md) |
| `ambientNow().temp` | Physics engine's diurnal/seasonal curve | Outdoor air temperature at the condenser — now used by ALL modes (§4d) via `condenserApproach()`/`effectiveCOP()` for both COP and the high-head-pressure safety cutout, not just VFD |

**Note on what real controllers do NOT get:** none of the three current modes read
`zoneTemp[i]` (product core temperature) directly — only air temperature. This is
intentional and matches real refrigeration control: the compressor reacts to the air/suction-side
sensor it has, not to a probe buried in the product (most facilities don't have one per pallet).
Product temperature is a *consequence* of how well air temperature is held, not a direct input to
these three modes. The planned AI mode is explicitly allowed to use more than this (see below) —
that's the point of comparing it against these three fairly.

## Shared outputs (all modes produce these)

| Output | What it drives in the simulation | Real-world actuator it stands in for |
|---|---|---|
| `state.compressorOn[i]` (bool) | Whether `stepZone` applies `coolingW` this tick | Contactor/relay coil energized → compressor motor runs |
| `state.capacityFraction[i]` (0–1) | Scales `coolingW` and, in VFD mode, the part-load efficiency penalty | 0/1 for a fixed-speed compressor; 0.0–1.0 commanded speed for a VFD/digital-scroll drive (via 0–10V, 4–20mA, or a fieldbus speed reference) |

**Shared safety layer (§4d, new):** before any mode-specific control law runs, `stepCompressor()` now
checks a high-head-pressure cutout common to all three modes — if condensing temperature (ambient +
condenser approach) exceeds 55°C, the compressor is forced off for a 5-minute cooldown regardless of
what the controller wants, exactly like a real high-pressure switch overriding the thermostat/BMS. A
fixed-speed condenser fan (Modes A/B) is structurally closer to this limit than VFD's floating head
pressure — a realistic emergent difference, not a mode-specific rule. State: `hpTripRemain[i]`.

---

## Mode A — Two-position (on/off relay), `cfg.mode === "rule"`

**Inputs used:** `airTemp[i]`, `target`, `band`, plus two internal timers (`minRunRemain[i]`,
`minRestRemain[i]`) that only depend on elapsed time since the compressor's own last transition —
no external input.

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
- Multiple staged compressors per zone (lead-lag) — would require restructuring the per-zone
  `compressorOn[i]`/`capacityFraction[i]` scalars into per-unit arrays throughout the control,
  physics, twin, and UI layers — a genuine rework, not a localized fix.
- Locked-rotor inrush as a real current-vs-time profile (rather than a flat one-time kWh charge).
- Adaptive-band memory across days (day-of-week/hour learned patterns).
- Contactor/relay electrical wear as its own failure mode, distinct from the compressor capacity
  wear §4d already added from cycle count.
