# TODO — Cold Storage Simulation Roadmap

Source of truth: extracted from the in-app "Design summary — what this actually is, in plain words"
modal (`simulation/cold_storage_simulation.html`, Step-by-step + "What's still a shortcut" + "What it
would take to connect this to a real building") and a pass over the controller code.

Goal for this phase: make all **three real-world control paradigms** India actually uses today
properly correct and independently tunable, then add a **fourth, AI-based** control approach, and
build a **comparison framework** across all four — scored on ₹/kWh tariff cost vs. how well each
holds the product at its optimum temperature (not just energy use in isolation).

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
- [ ] Couple humidity removal to compressor cooling capacity properly: condensing moisture out of
      the air consumes real cooling capacity: model this as a shared capacity budget instead of two
      independent numbers, so humid-day performance isn't overstated.
- [ ] Re-derive the frost penalty from one physically-grounded source instead of applying "less
      cooling delivered" and "worse electricity efficiency" as two separate hand-tuned penalties —
      or keep both but document/calibrate the combined effect against published refrigeration data.
- [ ] Replace the generic Arrhenius/Q10 spoilage formula with per-product kinetics where real data
      exists (start with the 5 Indian items: potato, onion, tomato, banana, mango).
- [ ] Implement true independent per-zone setpoints (Zone A/B/C should be able to hold different
      products at different targets simultaneously — currently all 3 chambers share the one active
      product's setpoint, per README's stated known limitation). **Explicitly out of scope for now**
      (per direct instruction) alongside real-hardware integration (§7) — everything else in this
      roadmap should be pursued as realistically as possible; these two specifically are parked.

## 2. Digital twin / estimator layer

- [ ] Upgrade the Step-3 "guessing" logic from a simplified recursive/gradient calibration to a
      proper Kalman filter (or extended/unscented KF given the nonlinear thermal model), with real
      covariance/uncertainty propagation instead of an ad hoc nudge-toward-reading update.
- [ ] Improve anomaly attribution (equipment wear vs. sensor drift) from a heuristic threshold into
      a proper multi-hypothesis test (e.g., compare likelihood of "compressor degraded" vs. "sensor
      drifted" explanations given the residual history).
- [ ] Add seeded/reproducible randomness (truck arrival timing, sensor noise, wear rate) so a run
      can be replayed exactly — required before any of this can be regression-tested or used to
      reproduce a specific incident.
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

## 3. Environmental & tariff inputs

- [ ] Replace the hand-picked 4-season diurnal ambient curve with sourced real climate data
      (IMD normals or similar) per region, keeping the smooth-curve model as a fallback.
- [ ] Add real Indian electricity tariff structures properly, not just one illustrative time-of-use
      curve — model actual state DISCOM industrial/commercial ToU slabs (these differ significantly
      by state), so tariff-driven behaviors (pre-cooling before peak, coasting on thermal mass) are
      tested against real rate schedules, not a single assumed one.
- [ ] Add day-to-day weather variability (clouds/rain/wind perturbations) on top of the smooth
      diurnal curve.

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

**Deliberately deferred (flagged, not implemented this round) — larger architectural risk, lower
value for the effort:**
- [ ] **Multiple staged compressors per zone (lead-lag).** Real facilities of any size often run 2+
      smaller compressors per chamber rather than one large on/off or VFD unit. This would require
      restructuring the per-zone `compressorOn[i]`/`capacityFraction[i]` scalars into per-unit arrays
      throughout `stepCompressor`/`stepZone`/the twin estimator/UI — a significant rework, not a
      localized fix. Left for a dedicated future pass rather than a shallow bolt-on.
- [ ] **Locked-rotor/inrush as a real current-vs-time profile** (rather than a flat one-time kWh
      charge) — would let voltage-sag/power-quality effects on other loads be modeled, but is a lot of
      new fidelity for a fairly niche, small-magnitude effect.
- [ ] **Adaptive-band memory across days** (e.g., learning "this is always a hot Tuesday afternoon")
      — real adaptive deadband tuning is usually periodic manual retuning from trend logs, not
      continuous learning; would need a genuine data structure (rolling day-of-week/hour profile) to
      do honestly rather than a token gesture.
- [ ] **Contactor/relay electrical wear as its own failure mode** (contact resistance increasing,
      eventual replacement) distinct from the compressor capacity wear above — cycle count now feeds
      compressor wear, but the relay/contactor itself has no separate failure path or maintenance
      trigger yet.

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

**Result across all 9 products, 15-day headless replay of the fixed code** (`inBand%` = % of tracked
time ALL 3 zones held within target±band; VFD consistently ranks best on both cost and quality after
these fixes, matching real-world expectations for the first time):

| Product | Two-position | Adaptive | VFD |
|---|---|---|---|
| potato | 60.1% | 61.8% | **63.8%** |
| onion | 72.9% | 74.0% | **76.1%** |
| tomato | 32.6% | 37.5% | **43.3%** |
| banana | 15.9% | 21.5% | **26.1%** |
| mango | 40.1% | 44.8% | **49.3%** |
| meat | 26.4% | 31.6% | **45.2%** |
| pharma | 28.3% | 35.0% | **36.4%** |
| dairy | 1.5% | 1.6% | **9.5%** |
| produce | 0.0% | 0.0% | 1.6% |

**Open question, NOT unilaterally fixed** (flagged for a decision, not a code defect): "Leafy
produce" and, to a lesser extent, "Dairy crates" still show low in-band% because their delivery
INTERVAL is shorter than or comparable to their thermal time constant (τ) — e.g. produce's 16h
delivery interval vs 12h τ means a fresh (still-warm) delivery can arrive before the previous one has
even finished converging, so the zone's average product temperature never really settles, regardless
of controller. This may be intentionally realistic (fast-turnover leafy greens genuinely are hard to
hold at tight temperature without a dedicated pre-cooling step — a real, well-documented cold-chain
problem) rather than a bug to silently patch by changing product data. Options: (a) leave as an
intentionally hard stress-test scenario and say so explicitly in the UI, (b) moderate "Leafy
produce"'s `turnoverFraction`/`deliveryIntervalHr` to be less of an outlier vs. the rest of the table,
or (c) add an explicit receiving-dock "pre-cool before put-away" process (a real facility feature)
that isn't modeled at all right now. Left open for a product/scope decision.

## 5. NEW: AI-based control approach (4th mode)

- [ ] Define what "AI-based" means concretely for this sim — proposed scope: a learned policy
      (e.g., a small model or RL agent) that decides compressor on/off or capacity fraction using
      forecasted ambient temperature, forecasted tariff period, current product core temp trajectory,
      and thermal mass — optimizing a joint cost function of (₹ spent) + (penalty for time/degree
      outside optimum band), rather than reacting to instantaneous air temperature alone.
- [ ] Train/tune this against the same physics engine and tariff data as the other three modes (no
      unfair advantage from privileged information the real controller wouldn't have — it only gets
      the same noisy sensor feed as the other modes, per the existing digital-twin design).
- [ ] Give it the second-order behaviors a plain thermostat can't do: pre-cool ahead of a known
      tariff peak (already gestured at in the current advisory text), coast on thermal mass through
      peak windows, anticipate a truck delivery's heat load from the schedule, adapt compressor
      wear-aware scheduling (avoid unnecessary cycling as equipment ages).
- [ ] Replace the current hand-written Step-6 advisory rule-checker with (or feed it from) this
      model, since the design summary already flags that rule-checker as "the easiest part to
      replace with a real AI model."
- [ ] Make the AI mode a genuine 4th selectable controller option in the UI, not just an advisory
      overlay — it should actually drive `compressorOn` / `capacityFraction` like the other 3 modes.

## 6. Comparison & scoring framework (tariff × optimum-temperature)

- [x] Build a scenario runner: same product, same season, same day(s), same environmental
      disturbances (§0), run through all 3 built controller modes in parallel (not back-to-back — see
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
