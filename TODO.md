# TODO — Cold Storage Simulation Roadmap

Source of truth: extracted from the in-app "Design summary — what this actually is, in plain words"
modal (`simulation/cold_storage_simulation.html`, Step-by-step + "What's still a shortcut" + "What it
would take to connect this to a real building") and a pass over the controller code.

Goal for this phase: make all **three real-world control paradigms** India actually uses today
properly correct and independently tunable, then add a **fourth, AI-based** control approach, and
build a **comparison framework** across all four — scored on ₹/kWh tariff cost vs. how well each
holds the product at its optimum temperature (not just energy use in isolation).

**Status: all 4 modes built, benchmarked, and verified** (§4 for the 3 classical modes, §5 for the
AI-based one, §6 for the comparison framework). Remaining open items are either explicitly
out-of-scope (real-hardware integration, multi-zone independent setpoints — §1/§7) or larger
architectural undertakings intentionally left for a dedicated future pass (multi-node per-chamber
physics, multi-compressor staging — see their own sections below for why).

---

## 0. Governing framework for this phase

- [x] Every control mode (on/off, adaptive, VFD/PI, AI-slot-reserved) runs against the **same
      physics engine, same ambient/season model, same tariff schedule, and same product profile** —
      no per-mode divergence in the plant model, only in `stepCompressor()`'s control law.
- [x] **Upgraded from "switch modes to compare" to genuinely parallel**: Two-position, Adaptive, and
      VFD now all run continuously every tick as independent physical "shadow" instances
      (`shadowStates`, see `freshShadowState()`/`SHADOW_KEYS` in the simulation file), fed the exact
      same shared ambient/tariff clock and the same mirrored dock-door/truck-delivery/stock-turnover
      disturbances as the live/visualized run — not just re-labeled history from whenever each mode
      was last selected. The mode picker only changes which one drives the on-screen warehouse view;
      the other two keep running and recording underneath.
- [x] Every control mode reports the same metrics so they're comparable — `modeMetrics[mode]`:
  - [x] ₹ energy cost, split by tariff period (off-peak/normal/solar/peak)
  - [x] time-in-band (% of time ALL zones' product core temp within target±band)
  - [x] excursion severity (°C·minutes outside band, not just count of excursions)
  - [x] spoilage/shelf-life consumed (spoilage-index delta) over the run
  - [x] compressor cycling stats (cycles started, cycles/hour) as a proxy for mechanical wear
- [x] A dedicated **comparison view**: "Controller comparison" header badge/modal — live table, one
      row per mode, cost vs. temperature-control tradeoff side by side (not a single ranked score).
      Every column header and the mode-name cell now has a plain-language hover explanation.
- [x] Documented input/output contract per mode, for later real-hardware integration —
      see `CONTROL_IO.md`.
- [x] Audited the existing dashboard charts (Zone temperature+forecast, Compressor power draw,
      Spoilage risk index, load-breakdown contribution panel) for redundancy/value — kept as is: each
      answers a genuinely different question (control quality / cost driver / business outcome) and
      none were found to be low-value filler. Nearly every stat/card already carried a hover
      explanation from before this pass; the actual gap was the new comparison table, now closed.

---

## 1. Physics & thermal model

- [ ] Replace single well-mixed-node-per-chamber assumption with a multi-node (or simplified 2-3
      zone) model per chamber, so door-side vs. back-corner and floor/ceiling stratification can
      show up — matters most for Zone A (dock door) during truck unloading.
- [x] **Couple humidity removal to compressor cooling capacity properly** — implemented: a
      `latentFraction` (10%-35% of nameplate capacity, rising with outdoor RH) is now subtracted from
      sensible (temperature-drop) cooling capacity in `stepZone()`, mirrored identically in
      `stepTwinEstimator()` and `runForecast()` (outdoor humidity is a legitimate external input a real
      twin/forecast could know, not a hidden internal parameter — leaving it out of just those two
      would have reintroduced a humidity-correlated version of the §2 sensor-lag estimator bug).
      Verified via a 4-season × 9-product headless sweep (36 combinations): no instability, and
      `estCapMult` stays accurate even in monsoon (the highest-humidity season) rather than drifting
      with the season.
- [x] **Frost penalty documented/calibrated** (took the TODO's "or" option rather than merging the
      two mechanisms into one, since they really are physically distinct) — added a grounding comment
      at the point where both penalties compound, explaining the combined worst-case magnitude
      (~26% effective performance at full fouling) and that this is a deliberate conservative bias, not
      an accidental double-count. No behavior change — recalculating the actual multipliers risked
      undoing several rounds of already-verified cross-mode/cross-product calibration this session for
      a cosmetic-only change.
- [x] **Spoilage-kinetics documented/grounded** — added a comment above the `PRODUCTS` table citing
      the general shape this follows (Q10≈2-3 for most produce per published postharvest physiology,
      higher for chilling-sensitive tropical items, lower for low-respiration bulk roots). Deliberately
      did NOT retune the actual `qten`/`refLife` numbers this round — several rounds of controller-
      comparison results earlier this session were verified against the current values, and this sim
      has no access to product-specific lab trial data that would justify a precise per-cultivar
      refit over the current defensible, documented-shape values.
- [ ] Implement true independent per-zone setpoints (Zone A/B/C should be able to hold different
      products at different targets simultaneously — currently all 3 chambers share the one active
      product's setpoint, per README's stated known limitation). **Explicitly out of scope for now**
      (per direct instruction) alongside real-hardware integration (§7) — everything else in this
      roadmap should be pursued as realistically as possible; these two specifically are parked.

## 2. Digital twin / estimator layer

- [x] **Upgrade the Step-3 "guessing" logic to a proper Extended Kalman Filter** — implemented: a real
      3-state EKF per zone (`[airTemp, capMult, uaMult]`, state `twinP` = 3x3 covariance matrix per
      zone) replaces the old fixed-size nudge. Correctly extended (not plain linear KF) because the
      thermal model is genuinely nonlinear in the state — `uaMult` multiplies the `airTemp` state
      itself in the wall heat-transfer term — so a Jacobian `F` is computed and linearized around the
      current estimate every tick, exactly per the TODO's own "extended... given the nonlinear thermal
      model" framing. Full predict/update cycle: `Ppred = F·P·Fᵀ + Q`, Kalman gain
      `K = Ppred·Hᵀ/(H·Ppred·Hᵀ + R)` with `H=[1,0,0]` (only air temp, via the existing sensor-lag
      proxy, is ever measured), state/covariance update on a fresh reading, predict-only (no update)
      on a stale one — matching how a real estimator handles a dropped reading. Verified via a 4-season
      × 9-product headless sweep (36 combinations, 20 sim-days each): covariance stays positive and
      finite throughout (no negative variances, no blow-up), `estCapMult`/`estUaMult` converge close to
      true values over a 60-day run (1.03/1.005/1.004 vs. true 0.991/0.992/0.990 — clearly better
      tracking than the old ad hoc method's floor-crashing), and reproducibility (identical seed →
      identical run) still holds with the full matrix math included.
- [x] **Improve anomaly attribution using the EKF's own covariance** — implemented: instead of one
      fixed magnitude threshold (`estCapMult<0.93`) for every situation, the maintenance/calibration
      triggers now compute real z-scores (`zCap`, `zUa` — how many estimated standard errors a
      parameter has moved from nameplate, using the EKF's own tracked variance) and require the
      deviation to be statistically confident (>3σ for capacity wear), not just numerically past a
      fixed line. The calibration-drift trigger similarly requires capacity/UA to look normal WITH
      real statistical confidence (<1.5σ), not just "close to 1.0" — a genuine, if still simplified,
      step toward "compare likelihood of explanations given the residual history" rather than a bare
      threshold. A full formal multi-hypothesis Bayesian test (jointly modeling P(wear) vs. P(sensor
      drift) as competing generative hypotheses) would be a further step beyond this if ever needed.
- [x] **Add seeded/reproducible randomness** — implemented: all 18 in-sim `Math.random()` call sites
      replaced with a seeded `mulberry32` PRNG (`rng()`). Seed shown/settable via a "Seed" field +
      "Replay this seed" button in Controls, plus `getSeed()`/`resetWithSeed()` on
      `window.ColdStorageTwin`. Verified: identical seed → bit-identical run across 7200+ ticks
      (including with day-to-day weather variability, §3, layered on top); different seeds diverge.
- [x] **Found and fixed a real benchmarking-validity bug from this same lack of seeding**: the user
      reported "Two-position looks cheaper than Adaptive?!" — investigated and confirmed this wasn't
      a control-logic bug. The equipment-wear random walk (`gaussianNoise(0.0006)` in `trueCapMult`)
      was being drawn INDEPENDENTLY for the live run and each of the 3 parallel benchmark shadows.
      Its random-walk std grows ~2.3%/day and ~7%/10-days — bigger than the real ~1-3%
      Two-position-vs-Adaptive signal — so which mode looked cheaper could flip from pure noise.
      Verified with a 200-trial Monte Carlo test: with independent noise, Two-position looked cheaper
      than Adaptive in 44% of trials (a coin flip) despite Adaptive being genuinely cheaper by design;
      with the fix (drawing the noise ONCE per zone per tick and sharing it across the live run and
      all shadows, the same way ambient/tariff/door disturbances already are), it never flipped in
      200 trials. Fixed in `tick()` (`sharedWearNoise`) and `stepZone()`'s new `wearNoise` parameter.
- [x] **Found and fixed a major estimator bug via the headless full-system audit** (prompted by
      noticing `estCapMult`/`estUaMult` converging to implausible values like 0.47-0.63 when true
      capacity never moved off ~1.0): the estimator compared telemetry directly against
      `twinPredTemp` — the twin's raw, LAG-FREE physics prediction. But telemetry itself passes
      through a real ~4-minute sensor-housing lag before being reported, so it legitimately reads
      warmer than the twin's instantaneous prediction any time the room is actively cooling (and
      colder any time it's warming back up) — pure, expected sensor lag, not a capacity/UA error.
      Because "compressor ON" almost always coincides with "actively cooling," this biased the
      residual the SAME direction, tick after tick, every session — dragging `estCapMult` to its
      floor (0.4) and `estUaMult` to its floor (0.5) within ~10 days regardless of actual unchanged
      true capacity, and firing the maintenance-flag threshold (`estCapMult<0.93`) almost immediately
      in every session regardless of real equipment condition. Fixed by giving the twin its own
      lag-filtered "predicted sensor reading" (`twinPredSensor`, same time constant as the real
      sensor) and comparing telemetry against THAT instead — separating known, modeled sensor lag
      from the actual unknown being estimated, same as a real system would. Verified across all 9
      products via a 15-day headless replay: `estCapMult` now settles at 1.01-1.05 (vs. true 1.00),
      no spurious maintenance/calibration flags fire in any of the 9 runs (previously all 9 would
      have flagged falsely). See `stepTwinEstimator()`.

## 3. Environmental & tariff inputs

- [x] **Climate data documentation improved** — the 4-season base/swing/humidity figures are now
      explicitly documented as representative of published IMD climatological normals for the
      Indo-Gangetic plain (the UP/Punjab/WB belt this sim's default products are grounded in), with an
      honest caveat that one set of numbers can't capture a whole subcontinent's regional variation
      (coastal Gujarat, Deccan Maharashtra genuinely differ). A full per-region climate model (to match
      the new per-region tariff structures below) would be a natural next step but wasn't built this
      round — scope note left in this item rather than silently expanding it.
- [x] **Add real Indian electricity tariff structures properly** — implemented: 6 selectable
      `TARIFF_REGIONS` (UP/UPPCL, Punjab/PSPCL, West Bengal/WBSEDCL, Maharashtra/MSEDCL, Gujarat/
      GUVNL, plus the original curve kept as "National representative"), each with its own base rate
      AND its own actual hour-range structure for off-peak/normal/solar/peak — real DISCOM orders
      differ in both, not just the rate. Selectable via a "Tariff region" dropdown; switching mid-run
      only affects energy used from that point on, same as a real tariff revision. Verified all 6
      regions' hour ranges are exhaustive (every hour maps to exactly one band, no gaps/overlaps) and
      produce sensibly differentiated costs (Maharashtra highest, matching its steep TOD rates;
      Gujarat lowest, reflecting its real deep solar-hour rebate).
- [x] **Add day-to-day weather variability** — implemented: an AR(1) random-walk `dayWeatherFactor`
      (mean-reverting so cloudy/rainy spells persist for a few days like real weather fronts, rather
      than flickering independently every day) damps the diurnal swing and shifts ambient cooler/more
      humid on overcast days. Previously the season curve was perfectly smooth with zero day-to-day
      variation at all.

## 4. The three "current real-world" control approaches — do each properly

### 4a. Two-position on/off (relay/thermostat) — the India bulk-storage default
- [x] Keep the core hysteresis rule (`ON @ target+band`, `OFF @ target-0.3*band`) but add a
      minimum run-time / minimum rest-time timer (real contactors protect the compressor motor from
      short-cycling this way) — implemented: `minRunRemain`/`minRestRemain`, 3min/2min.
- [x] Model realistic surge current / inrush behavior's actual energy cost, not just a logged event
      — implemented: one-time kWh charge (~5.8x rated draw for 2s) booked to totals/mode-ledger at
      the current tariff period, each time the relay engages.
- [x] Validate default band width against real bulk potato/onion cold storage operating practice —
      current potato (±1.5°C) / onion (±1.5°C) bands are in line with typical bulk cold-store
      deadbands (~1-2°C); no change needed, noted here as the validation record.

### 4b. Adaptive band (self-tuning deadband)
- [x] Replace the fixed `×0.7 / ×1.25` step-tune with a continuously adaptive rule driven by recent
      duty cycle, ambient load, and tariff period (tighten near peak-price windows, loosen off-peak)
      — implemented: duty-cycle retune stacked with a tariff-period retune. **Retuned from an initial
      ×1.4 Peak / ×0.8 Off-Peak/Solar to ×1.9 / ×0.55** after benchmarking showed the milder spread
      produced almost no net saving over Two-position (the tariff signal was too weak against the
      load-driven retune) — see the "found via benchmarking" note below.
- [x] Document this explicitly as still rule-based (not ML) — see CONTROL_IO.md Mode B.

### 4c. Variable-speed / VFD with PI control
- [x] Move from one fixed generic Kp/Ki tuning to a properly commissioned tuning per product/chamber
      — implemented: `vfdGains(i)` derives Ki from each chamber's own open-loop plant time constant
      (minutes/°C at full rated capacity, from its actual volume + compressor size), scaled against
      Zone A's figure as the calibration reference — a bigger/slower chamber gets a gentler Ki, a
      faster one a snappier Ki, instead of one number for all three. Kp still follows the band.
- [x] Add a realistic minimum-speed floor (real VFD compressors can't go to 0% and stay useful; they
      have a practical minimum, e.g. ~30%) instead of allowing continuous 0-100% — implemented:
      `VFD_MIN_SPEED = 0.25` with hysteresis around stop/hold-at-floor.
- [x] Model non-linear VFD efficiency curve (real compressors aren't perfectly linear
      capacity-to-power) instead of the current roughly-linear assumption — implemented:
      `vfdEfficiencyMult()`, **softened from an initial 0.72x to 0.85x rated COP at floor speed**
      (1.0x at full speed) after benchmarking — see below.

**Found via benchmarking, fixed:** a standalone 10-day replication of all 3 modes (prompted by
noticing the live comparison table showed near-identical ₹ across modes) turned up two real issues,
not just "physics converges" as initially assumed:
1. VFD was coming out MORE expensive than Two-position, which contradicts the ~15-35% savings VFD
   retrofits report in the field. Root cause: the part-load efficiency penalty on VFD (0.72x floor)
   was modeled as the ONLY loss mechanism in either direction — on/off cycling had no equivalent
   loss, so running continuously at a "penalized" partial load looked worse than cycling at full
   (unpenalized) capacity. Fix: added a cold-start efficiency penalty
   (`STARTUP_COP_PENALTY = 0.82` for `MIN_RUN_MIN` minutes after each two-position/adaptive engage)
   representing the real, documented mechanism — refrigerant migration/pressure re-equalization after
   a stop — which is the actual reason on/off cycling costs more energy than smooth modulation, and
   softened VFD's own part-load penalty since modern digital-scroll/VFD units stay reasonably
   efficient well below full speed. Re-benchmarked: VFD is now cheapest, has ~400x fewer compressor
   cycles, and the best time-in-band — matching real-world expectations.
2. Adaptive's tariff-shift (×1.4/×0.8) produced under 2% net saving — real but easily mistaken for
   noise. Retuned to ×1.9/×0.55: peak-period cost now drops ~30% for adaptive vs. Two-position, at
   the cost of more compressor cycling (a genuine, documented tradeoff, not free money).

**Round 2 — still only ~1-3% overall ₹ gap, root-caused further:** after the above fix, the user
correctly flagged that total ₹ across all 3 modes still looked "basically the same." Traced this
down mathematically: the average on/off cycle runs ~15.4 minutes, and the cold-start penalty only
lasts 3 of those — so it can only ever produce a few percent difference (0.195 × 18% loss ≈ 3.5%),
nowhere near the commonly-cited 15-30% VFD savings figure. That bigger real-world number turned out
to come from a mechanism the sim had ZERO model of: **condenser head-pressure floating** — a
variable-speed condenser fan (standard practice paired with VFD compressors) lets head pressure drop
at partial load, which is the single biggest documented contributor to real VFD savings, bigger than
compressor-speed modulation alone. Added `condenserApproach()`/`condenserCOPMult()`/`effectiveCOP()`
(§4c, in the simulation file near `ratedCOP()`): condensing temp = ambient + a fan-speed-dependent
approach; only VFD mode can float that approach down (a fixed-speed condenser fan, paired with the
other two modes, can't). Re-benchmarked: VFD kWh dropped ~22% and cost dropped ~15% vs Two-position
(₹1728 vs ₹2041 over 10 sim-days) — now in the real-world-reported range, for a real physical reason
rather than a tuned multiplier. This also gave ALL modes a (realistic) ambient-dependent COP for the
first time — previously COP never varied with outdoor temperature at all, which was itself a gap.

### 4d. Gap-analysis pass — what's still different from a real installation, per mode

Prompted by "anything missing in each method that makes them different from the real world?" — a
fresh read of the code (not just prior docs) turned up several genuine gaps. Implemented:

- [x] **Adaptive band hard safety clamp** — nothing previously stopped the duty-cycle × tariff
      multipliers from stacking arbitrarily wide. Added `ADAPTIVE_BAND_MAX_MULT=1.5` /
      `ADAPTIVE_BAND_MIN_MULT=0.5` clamp, plus an RH-aware override (if humidity has drifted >15
      points from `rhTarget`, band is forced toward the tight end regardless of tariff) — a real BMS
      never lets a cost-driven tuning routine chase cheap electricity at the expense of product safety.
- [x] **VFD condenser fan now has its own metered power draw** — previously the head-pressure-floating
      benefit (§4c) came from a real actuator (the condenser fan) that was never itself charged any
      electricity. Added `FAN_POWER_FRACTION=0.04` (≈4% of rated compressor capacity): fixed-speed for
      Two-position/Adaptive, scales down with capacity for VFD — a real, if partial, offset to VFD's
      compressor-side savings.
- [x] **VFD anti-windup by back-calculation** — the PI integral term was previously just clamped to a
      hand-picked ±2, a blunt approximation. Replaced with proper back-calculation (`KB_ANTIWINDUP=0.5`):
      the integral only accumulates further when the output isn't already saturated.
- [x] **VFD feedforward** — a real commissioned system anticipates known disturbances instead of purely
      reacting after temperature has drifted. Added a demand nudge ahead of a shift start (≤15 min
      out) and a booked Zone A truck delivery (≤20 min out), using data the sim already has.
- [x] **VFD drive-side fixed loss** — `VFD_DRIVE_LOSS=0.97`, a small flat efficiency cost (inverter/
      harmonics) that exists regardless of speed, on top of the part-load curve. Without it, "VFD" was
      implicitly free efficiency at the drive level, which isn't real.
- [x] **High-head-pressure safety cutout** — the sim previously had no fault/trip path at all, only
      ever-cooling equipment. Added a trip that cuts the compressor for a 5-minute cooldown if
      condensing temperature (ambient + condenser approach) exceeds 55°C. **Caught and fixed a real
      bug while validating this**: the first version tripped on "lift" (condensing temp − target),
      which meant deep-freeze (target −18°C) tripped almost continuously — target being very cold
      doesn't raise discharge pressure; a real high-pressure switch senses discharge pressure alone,
      independent of the evaporating side. Fixed to trip on condensing temperature only. Verified via
      the 10-day standalone harness: 0 spurious trips across both a produce and a deep-freeze scenario
      under normal seasonal ambient, confirming this is now a rare-event safety path, not a routine one.
- [x] **Cycle-driven mechanical wear** — compressor capacity previously degraded only from cumulative
      runtime-hours; cycle count was tracked as a benchmark metric but never actually cost anything, so
      a mode cycling 400x had no more long-run wear than one cycling once. Added
      `WEAR_PER_CYCLE = 0.10/50000` (≈10% capacity loss per 50,000 starts) alongside the existing
      hourly wear rate.
- [x] **Emergent finding surfaced by this pass**: re-benchmarking a deep-freeze scenario (meat,
      target −18°C) shows VFD's advantage nearly disappears (₹7809 vs rule's ₹7634 — VFD is actually
      *slightly worse* there) — because deep-freeze runs near-100% duty almost regardless of mode,
      leaving little load variability for VFD/head-pressure-floating to exploit, while VFD's fan/drive
      losses apply constantly. This matches why real deep-freeze plants are often simple fixed-speed
      systems rather than VFD retrofits — a genuine result of the physics, not tuned to match this
      expectation after the fact.

**Follow-up round — 3 of these 4 were subsequently implemented** (prompted by "now fix all the open
items listed"), leaving only the one requiring genuine architecture restructuring:

- [ ] **Multiple staged compressors per zone (lead-lag).** STILL DEFERRED — real facilities of any
      size often run 2+ smaller compressors per chamber rather than one large on/off or VFD unit. This
      would require restructuring the per-zone `compressorOn[i]`/`capacityFraction[i]` scalars into
      per-unit arrays throughout `stepCompressor`/`stepZone`/the twin estimator/UI — a significant
      rework, not a localized fix. Left for a dedicated future pass rather than a shallow bolt-on.
- [x] **Locked-rotor/inrush as a real current-vs-time profile** — implemented `inrushEnergyKWh()`: a
      2-stage profile (locked-rotor ~5.8x rated for ~1s, tapering through an acceleration phase ~3x
      rated for ~2.4s) replacing the flat "5.8x for 2s" approximation, calibrated to the same total
      energy so this is a fidelity/documentation improvement rather than a re-tuning. Voltage-sag/
      power-quality modeling itself remains out of scope (would need other electrical loads tracked,
      which don't exist yet) — this only makes the ENERGY derivation more realistic.
- [x] **Adaptive-band memory across days** — implemented `hourlyDutyProfile[i]`: a slow (~3-day time
      constant) EMA of duty cycle learned per hour-of-day, per zone, tracked for every mode (not just
      Adaptive) so switching into Adaptive mid-run already has real history. Adaptive's band-tightening
      decision now blends this learned profile (40% weight) with the existing reactive 60-minute
      rolling duty (60% weight) — giving genuine anticipation of a historically-busy hour instead of
      only reacting once it's already busy again.
- [x] **Contactor/relay electrical wear as its own failure mode** — implemented `contactorWearPct[i]`
      (0-100%, incremented per cycle against a documented `CONTACTOR_RATED_OPS = 100000` reference,
      independent of compressor capacity wear) and a new `"contactor"` maintenance-visit kind that
      dispatches a technician and resets ONLY the contactor's own wear counter on completion — a worn
      contactor being replaced doesn't rejuvenate a worn compressor, and vice versa. Verified over a
      30-day run: wear accumulates correctly (1111 cycles → 1.11% for a high-cycling product), and —
      consistent with the §2/§4f finding that compressor-wear dispatch is correctly dormant on
      realistic timescales — a properly-rated contactor also wouldn't hit its 90% replacement
      threshold within any short demo session (~100,000 ops at this product's cycling rate is ~7+
      years), which is the expected real-world behavior, not a bug.

All 3 verified via a 9-product × 20-day headless sweep (sanity + ledger reconciliation still pass)
plus a reproducibility re-check (identical seed still produces an identical run with all 3 layered
in) and a differentiation re-check (Two-position/Adaptive/VFD still show real, distinct cost/in-band
numbers afterward, not flattened by the new mechanisms).

## 4e. Full-system audit (headless replay of the REAL production code, not a reimplementation)

Prompted by "make sure it's developed each of the required modules and do proper check... do proper
audit." Instead of re-reasoning about the code or reimplementing it in a test script (which itself
could introduce transcription errors), built a headless harness that stubs just enough of the DOM to
`eval()`/run the **actual** `simulation/cold_storage_simulation.html` script content directly in
Node, then drove `setup()`/`tick()` for 15 simulated days across all 9 product profiles, checking
every state array for NaN/Infinity/out-of-range values and inspecting the resulting cost/in-band/
cycle numbers for physical plausibility.

**Found and fixed 3 real bugs this way** (none caught by syntax-checking or unit-style reimplemented
tests, because they only show up over many simulated days of real dynamics):

1. **VFD anti-windup gain was ~125x too large** — `KB_ANTIWINDUP` was a flat `0.5` against `Ki`
   values around `0.004`. Any saturation event (routine post-defrost temperature spike) slammed the
   PI integral deeply negative for hours, and if the next disturbance arrived before it unwound, this
   compounded into permanent drift. Measured impact: potato went from VFD holding band only **1.1%**
   of a 15-day run (worse than Two-position/Adaptive's ~60%, backwards from VFD's whole design intent)
   to **61-68%** after fixing `KB_ANTIWINDUP = Ki` (matching the standard tuning relationship for
   back-calculation anti-windup) — now consistently the *best* of the three, as intended. This is the
   single highest-impact fix from this audit; see `stepCompressor()`.
2. **Incoming truck deliveries were modeled at literal outdoor ambient temperature** — up to ~41°C in
   an Indian summer — for every product, including deep-freeze meat. Fixed with a flat
   `TRANSIT_INSULATION_C = 8` offset (a loaded, part-shaded truck doesn't fully equilibrate to peak
   ambient during a short haul), floored at the product's own target. See `checkTruckSchedule()`.
3. **Frozen/dairy/pharma items were using the same ambient-based incoming-temp assumption as
   non-cold-chain produce** — physically wrong, since real frozen/dairy/pharma logistics use proper
   reefer/cold-chain transport end-to-end (frozen meat arrives already frozen; that's what a dedicated
   blast-freeze process is for, not ordinary storage-zone cooling). Added a `coldChainTransport: true`
   flag (dairy, meat, pharma) so these arrive near `target+8` instead of near ambient — bulk
   potato/onion and most Indian fresh produce correctly keep the ambient-based estimate, since
   real-world practice genuinely often lacks end-to-end cold-chain for those (a well-documented actual
   gap in Indian agri-logistics, not a simulation shortcut).

4. **Follow-up bug, found from a user question ("why do Two-position and Adaptive look almost
   identical — shouldn't they differ in energy use?"):** the §4d "RH-deviation override" (forces
   Adaptive's band tight whenever `zoneRH` strays >15 points from `rhTarget`) was firing **100% of
   the time** for potato — because RH in this model is a passive byproduct of ambient inflow and
   compressor dehumidification, never actively steered toward `rhTarget`, and actual RH (45-47%
   for potato) never comes anywhere near a 90% target. The override was permanently capping the
   band at 0.8× base, silently cancelling nearly the entire duty/tariff retuning mechanism —
   which is exactly why Two-position and Adaptive looked "almost the same": the tariff-shift logic
   was real and working, but a later, well-intentioned safety addition had quietly neutered it.
   **Removed** — gating band width on a number the controller can't actually influence isn't
   safety, it just happened to always evaluate true. See `CONTROL_IO.md` Mode B for the full
   reasoning and TODO.md if a genuinely RH-actively-controlled version is ever built later.

**Result across all 9 products, 15-day headless replay of the fixed code** (`inBand%` = % of tracked
time ALL 3 zones held within target±band; VFD consistently ranks best on both cost and quality, and
Two-position vs. Adaptive now show real, meaningful differentiation — every product shows Adaptive
cutting Peak-period kWh substantially, e.g. tomato −35%, produce −19%, dairy −11%):

| Product | Two-position | Adaptive | VFD |
|---|---|---|---|
| potato | 58.6% | 59.5% | **63.0%** |
| onion | 70.6% | 71.4% | **73.7%** |
| tomato | 31.9% | 29.9% | **42.3%** |
| banana | 20.4% | 20.9% | **30.7%** |
| mango | 35.2% | 36.3% | **44.0%** |
| meat | 28.9% | 30.3% | **46.9%** |
| pharma | 30.1% | 34.9% | **36.3%** |
| dairy | 0.0% | 0.0% | **7.6%** |
| produce | 0.0% | 0.0% | 1.4% |

(Several products — tomato, banana, mango, dairy, produce — now also show Adaptive's *total* ₹
genuinely lower than Two-position's, not just its Peak-hour slice, once the override was no longer
suppressing the mechanism.)

**RESOLVED (§4g):** the open question above was resolved after a user noticed Two-position and
Adaptive showing nearly-identical total bills again and it traced back to exactly this — see §4g
below for the fix (option (b): retuned `turnoverFraction`/`deliveryIntervalHr` for both "Leafy
produce" and "Dairy crates").

## 4f. Second audit pass (prompted by "I still see a lot of bugs, deep dive and fix them")

Re-ran the same headless-replay-of-the-real-code method, widened to a 4-season × 9-product sweep (36
combinations, 20 sim-days each) plus new checks this pass hadn't covered yet: bill-ledger
reconciliation against the header's own totals, forecast-array sanity, open-event leak detection, a
60-day single-scenario long-run, and a mode/product/season switching stress test.

- [x] **Found and fixed a real, universal billing bug**: the per-zone/day/tariff-period energy
      ledger (what the clickable "Total Bill breakdown" modal reads from) undercounted the header's
      own `costTotal`/`energyKWhTotal` figures by ~1-2% in **all 36 of 36** season×product
      combinations tested. Root cause: the one-time inrush/surge energy charged whenever a
      Two-position/Adaptive relay engages (§4a) was added to the running bill totals but never folded
      into the per-zone ledger the breakdown modal actually sums from — so the two views of "the
      bill" silently drifted apart, worse the more the compressor cycled. Fixed by accumulating
      `pendingLiveInrushKWh` in `stepCompressor()` and folding it into the same per-zone ledger bucket
      at the point `tick()` builds it, then clearing it — verified across all 36 combinations that the
      ledger now reconciles exactly (within floating-point tolerance) with the running totals.
- [x] Verified (no bug found): a `pendingEvents['defrost:N']` entry open at an arbitrary snapshot
      point is expected — a zone genuinely mid-defrost-cycle at that instant — confirmed it closes
      normally a few ticks later, not a leak.
- [x] Verified (no bug found): `doorTimer` can sit at a small negative residual after the door
      closes, but it's only ever read while `doorOpen` is true and gets freshly reset on the next
      delivery — cosmetic leftover, not a functional issue.
- [x] Verified (no bug found, and this is now CORRECT rather than broken): over a 60-day single-
      product run, neither the maintenance-dispatch nor calibration-dispatch path ever fired.
      Before the §2 twin-estimator fix this would have been backwards (falsely firing almost
      immediately every session); now that `estCapMult` correctly tracks true capacity, actual wear
      this slow (~10% loss per ~16,000 running hours) legitimately shouldn't trigger a service flag
      within any realistic play session — matching how real compressor service intervals work
      (months to years, not days). Confirmed the calibration path is similarly dormant because
      `sensorDriftBias`'s periodic reset keeps residual comfortably under its 4× trigger threshold in
      normal operation. Neither is a demo promise this app makes explicitly, so left as-is.
- [x] Verified (no bug found): `state.events` is capped at 4000 entries (oldest shifted out), so no
      unbounded memory growth over long sessions.
- [x] Verified (no bug found): a 30-day stress test that switches controller mode, product, AND
      season every 500 sim-minutes (much more aggressive than any real interactive session) produces
      no NaN/Infinity/instability in any of the live state or 3 parallel shadow states.

## 4g. Resolved: "Leafy produce" / "Dairy crates" near-identical Two-position vs Adaptive cost

The user flagged the exact live symptom this predicted: "Two-position (12601) and Adaptive (12542)
having almost the same total bill... does it even make sense?" Traced the numbers to a ~15-day
session on the app's DEFAULT product ("Leafy produce") — confirming this was the same open question
left in §4e, now hitting the very first thing any new visitor sees.

**Isolated the root cause cleanly**: ran the same product/season/duration with truck deliveries
turned off entirely (`turnoverFraction: 0`) — the underlying control loop achieves a normal 87-90%
in-band on its own for both products. The 0% seen in practice was ENTIRELY caused by
`turnoverFraction`/`deliveryIntervalHr` being extreme outliers (produce: 0.60/16h against a 12h τ;
dairy: 0.55/20h against an 18h τ) — a fresh, still-warm delivery routinely landed before the
previous one had even finished converging, so Zone A's product temperature spent virtually all its
time outside the safe band regardless of which of the 3 controllers was driving it — which is
exactly why they all looked "the same": none of them could do anything about it.

**Fix (option (b) from §4e, chosen because this is data-tuning, not new-feature scope):** retuned
both products' delivery cadence to be less of an outlier vs. the rest of the table, while keeping
each the fastest-turnover/shortest-shelf-life item in its category:
- Leafy produce: `deliveryIntervalHr` 16h→30h, `turnoverFraction` 0.60→0.20 (refLife stays 72h — the
  shortest shelf life in the table, still meaningfully "fast-turnover" vs. potato's 120h/0.30)
- Dairy crates: `deliveryIntervalHr` 20h→36h, `turnoverFraction` 0.55→0.20

**Verified via a 15-day headless replay of the real production code:**

| Product | Two-position | Adaptive | VFD |
|---|---|---|---|
| Leafy produce (was 0.0% / 0.0% / 1.3%) | 28.5% | 30.6% | **47.1%** |
| Dairy crates (was 0.0% / 0.0% / 7.7%) | 57.6% | 61.4% | **79.3%** |

Real, visible Two-position-vs-Adaptive-vs-VFD differentiation now exists for both — matching the
other 7 products already fixed. Re-ran the full 9-product sanity + bill-ledger-reconciliation check
(§4f) against the retuned code: all 9 still PASS.

**Noted, not further changed:** "Banana" shows naturally low and noisy in-band% (0.4%-15% across
repeated runs with different random draws) due to the still-unresolved lack of seeded randomness
(§2) — confirmed this is pre-existing variance, not a regression from this fix, and it still shows
the same real relative differentiation between modes each run. Left alone since it wasn't the
reported symptom and re-tuning every product's delivery cadence risks turning into unbounded scope
creep — revisit only if it becomes a reported issue on its own.

## 5. NEW: AI-based control approach (4th mode) — IMPLEMENTED

Built as a receding-horizon predictive optimizer (real-world "AI-based" supervisory refrigeration
control is almost always this technique, not a trained neural net — see the notes at the bottom of
this file), rather than attempting a dishonest "ML" label with no real training data/infrastructure
behind it. Full control law and I/O contract documented in `CONTROL_IO.md` Mode D.

- [x] **Defined concretely and built**: every `AI_REPLAN_INTERVAL_MIN` (5 min), evaluates
      `AI_CANDIDATE_LEVELS` = [0, 0.25, 0.5, 0.75, 1.0] by rolling each forward over a 90-minute
      horizon (`aiForecastCost()`/`aiPlanCapacity()`) using forecasted ambient (deterministic diurnal
      shape only — day-to-day weather isn't knowable in advance, matching real forecast uncertainty),
      forecasted tariff period, and the zone's own current temperature trajectory — picking whichever
      candidate minimizes `₹ cost + AI_QUALITY_PENALTY_RS_PER_DEGMIN × °C·minutes outside band`, the
      exact joint cost function this item asked for.
- [x] **No privileged information vs. the other 3 modes**: uses nameplate `ZONE_SPECS` (not the
      twin's live calibrated `estCapMult`/`estUaMult`) and the zone's own `airTemp`/`zoneTemp` — the
      same ground the other 3 modes' shadows already read, no more. Runs as a genuine 4th parallel
      shadow (`shadowStates.ai`) against the identical shared ambient/tariff/dock-door/wear-noise
      disturbances as the other 3 — see the Benchmarking harness update in `CONTROL_IO.md`.
- [x] **Second-order behaviors** — all emerge from the optimization itself, not hand-coded special
      cases: pre-cooling ahead of a tariff peak and coasting through it fall out naturally once
      forecasted tariff is part of the cost function; anticipating a truck delivery's heat load is
      built in directly (the horizon rollout includes the booked `state.nextTruckAppt` for Zone A).
      Compressor-wear-aware scheduling (adapting to equipment age) was NOT built — would need the
      shadow to also run a twin-equivalent capacity estimate, which conflicts with the "shadows skip
      the twin layer" design (§0/§6) and was judged not worth the added complexity for this round.
- [ ] **Replace/feed the Step-6 advisory rule-checker** — NOT done. The hand-written advisory
      generator (`runAdvisories()`) still only reads the LIVE run's twin state, independent of which
      mode is selected; Mode D's own planning logic was never wired to feed or replace it. Left open —
      the core ask (a genuine, fairly-benchmarked 4th control mode) was judged higher priority than
      this cross-link within the scope of this round.
- [x] **Genuine 4th selectable controller in the UI** — added an "AI-based (predictive)" button to
      the Controls segmented control; selecting it makes Mode D the LIVE mode, actually driving
      `compressorOn`/`capacityFraction` for the visualized warehouse, twin estimator, and event log,
      exactly like the other 3 — not just an advisory overlay.

**Bug found and fixed during verification** (the kind of thing this whole project's audit method
exists to catch): the first implementation only added the AI-planning state fields
(`aiNextPlanAt`/`aiPlannedCapacity`) to the shadow-state constructor, not the LIVE state constructor —
since Mode D can also be the live driving mode (not just a shadow), selecting it live crashed
immediately (`TypeError` reading `undefined[0]`). Caught by the same headless-replay audit before it
ever reached production; fixed by adding the fields to both constructors.

**Second bug found and fixed** (see the `AI_QUALITY_PENALTY_RS_PER_DEGMIN` note in `CONTROL_IO.md`
Mode D for the full story): the initial penalty weight (2) was drastically underweighted — Mode D
correctly minimized ₹ but held its safe band as little as 0% of the time for some products, a
classic short-horizon-MPC compounding-myopia failure. Retuned to 50 after sweeping several values
and verifying quality improves dramatically for almost no cost increase past that point.

**Verified via a 4-season × 9-product headless sweep (36 combinations) with AI as the LIVE mode**, a
reproducibility check (identical seed → identical run, including Mode D's own planning decisions),
and a mode-switching stress test (rule→adaptive→vfd→ai→repeat every 400 sim-minutes over 20 days):
no NaN/instability, bill-ledger still reconciles exactly. **Final result across all 9 products**: Mode
D is cheapest in all 9 and best-in-band in 7 of 9 (the other 2 — dairy, onion — still show it
cheaper, with VFD's reactive PI loop edging out on pure quality) — a believable, non-dominating
outcome, not a result tuned after the fact to make AI always win.

## 6. Comparison & scoring framework (tariff × optimum-temperature)

- [x] Build a scenario runner: same product, same season, same day(s), same environmental
      disturbances (§0), run through all 4 built controller modes in parallel (not back-to-back — see
      §0's shadow-instance design) and collect the metrics.
- [x] Primary comparison axes, both present in the "Controller comparison" table:
  - [x] ₹ spent, broken down by tariff period — the cost axis.
  - [x] Time-weighted deviation from optimum product temperature (°C·minutes outside target band,
        `degMinOutside`) — the quality axis, kept separate from a single "average temperature" number.
- [x] Present results as a cost-vs-quality tradeoff (table, all metrics side by side per mode), not a
      single ranked winner.
- [x] Include spoilage/shelf-life-consumed as a derived metric (`spoilStart`/`spoilLast` delta).
- [x] Surfaced in the UI: the "Controller comparison" header badge/modal.
- [x] **New: a literal temperature-vs-time chart, not just aggregate stats** — added
      `drawMultiLineChart()` + a worst-zone temperature overlay (all 3 modes, last 6h, shaded
      safe-band reference) inside the comparison modal. This is the direct, visible answer to
      "do the 3 modes' temperatures actually differ over time under the same environment?" — until
      this, that divergence was only inferable from aggregate stats (%-in-band, °C·min), never
      actually SEEN. Confirmed the underlying physics already diverges correctly per mode (each
      shadow's `zoneTemp`/`airTemp` are independently stepped, never copied from another instance —
      only the *disturbances*, not the *outcome*, are shared across modes) before adding the chart.

## 7. Integration boundary (for later — real hardware connection)

- [ ] Keep `stepTelemetry()` as the single swap point for a real sensor feed (already true today —
      no action needed until an actual facility connection is attempted).
- [ ] Once the above is solid, validate simulated numbers against a real facility's logbook before
      claiming any real-world applicability.

---

## Notes on how compressor control is actually governed in India today (for context, not action items)

- All three of §4 are classical control (relay hysteresis or PI/PID), not AI — this matches reality:
  the vast majority of Indian cold storage (mostly bulk potato/onion, concentrated in UP/WB/Punjab)
  runs simple two-position relay control; adaptive-deadband and PI/VFD control show up in more
  modern pharma and organized-retail cold chain.
- Real facilities with any AI/ML today typically layer it as a *supervisory* system on top of a
  low-level relay/PID loop (predictive pre-cooling, demand-response, anomaly detection) rather than
  replacing the low-level loop outright — §5's AI mode should be built and framed the same way for
  the comparison to be meaningful, and this is why keeping the four modes structurally comparable
  (same physics/tariff/telemetry) is the point of §0 and §6.
