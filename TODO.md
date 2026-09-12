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
      product's setpoint, per README's stated known limitation).

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

- [ ] Build a scenario runner: same product, same season, same day(s), same random seed, run through
      all 4 controller modes back to back (or in parallel state copies) and collect the metrics from
      §0.
- [ ] Primary comparison axes:
  - [ ] ₹ spent per day/week (broken down by tariff period) — the cost axis.
  - [ ] Time-weighted deviation from optimum product temperature (°C·hours outside target band) —
        the quality axis. Do not collapse this into a single "average temperature" number; excursion
        severity and duration both matter for spoilage.
- [ ] Present results as a cost-vs-quality tradeoff (e.g., a scatter of ₹/day against °C·hours
      outside band per mode), not a single ranked winner — different products/seasons may favor
      different modes.
- [ ] Include spoilage/shelf-life-consumed as a secondary derived metric from the temperature axis,
      since that's the real business consequence of poor temperature control.
- [ ] Surface this comparison in the UI (new modal or panel, alongside Events/Glossary/Design
      summary) so the tradeoff is visible to someone using the tool, not just logged internally.

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
