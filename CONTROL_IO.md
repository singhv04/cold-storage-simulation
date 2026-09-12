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
current tariff period's cost — a real, if small, ₹ cost of cycling, not just a logged event.

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
band *= (period == "peak") ? 1.4                 // tariff-based: loosen during Peak (coast on thermal mass)
        : (period in {"offpeak","solar"}) ? 0.8  // tighten during cheap hours (bank cooling ahead of Peak)
        : 1.0
```

**Output:** same shape as Mode A.

**Real-world equivalent:** this is still classical rule-based control, not ML — it matches a BMS
with a manually-programmed adaptive-deadband routine, or a technician who periodically retunes the
deadband from observed trends and the site's known tariff schedule. To connect: same as Mode A, plus
whatever field bus/database the real site's actual duty-cycle log and tariff calendar live in.

## Mode C — Variable-speed (VFD/PI), `cfg.mode === "vfd"`

**Inputs used:** `airTemp[i]`, `target`, `band`, plus the controller's own running integral term
`piIntegral[i]` (pure internal state, carried tick to tick — no external input beyond the error),
plus each chamber's own fixed physical size/capacity (`ZONE_SPECS[i]`) used once to derive its gains.

**Control law:**
```
over = airTemp[i] - target
{Kp, Ki} = vfdGains(i):
    Kp = 1/band                                              // same as before, error-driven
    minutesPerDegreeAtFullTilt = airCapacityKJperK(i)*1000 / compCapacityW[i] / 60
    Ki = 0.004 * (REFERENCE_MIN_PER_DEG=1.5 / minutesPerDegreeAtFullTilt)  // per-chamber commissioning, §4c
piIntegral[i] = clamp(piIntegral[i] + over*Ki*dt, -2, 2)
demand = clamp(over*Kp + piIntegral[i], 0, 1)
if 0 < demand < VFD_MIN_SPEED(0.25):             // §4c minimum-speed floor
    demand = (compressor was OFF) ? 0 : (demand < 0.125 ? 0 : VFD_MIN_SPEED)
capacityFraction[i] = slew-limited step toward demand, max 0.06/min
compressorOn[i] = capacityFraction[i] > 0.03
```

**Output:** `capacityFraction[i]` ∈ [0,1] continuous; `compressorOn[i]` derived from it.
Delivered power also folds in `vfdEfficiencyMult(capacityFraction)` — a non-linear part-load
efficiency curve (0.72× rated COP at the speed floor, 1.0× at full speed) — this is an energy-model
detail, not a controller input/output, but it means the same `capacityFraction` output costs more
₹/kWh-of-cooling at low speed than at high speed, which is realistic VFD behavior.

**Real-world equivalent:** a PI/PID loop running on a PLC or the compressor's own onboard
controller, driving a VFD/digital-scroll unit's analog or fieldbus speed reference. To connect: feed
the real `over` (measured − setpoint) into the same PI math (or better, hand this off to the site's
existing PID block and just read/write its output), and write `capacityFraction` out as the real
0–10V/4–20mA/fieldbus speed command instead of a JS number.

## Mode D — AI-based (planned, not yet implemented — see TODO.md §5)

**Inputs it is allowed to use that A/B/C are not** (this is the point of building it — a learned
policy can legitimately use more context than a bare thermostat can):
- The twin's own noisy telemetry feed and forecast (`state.telemetry`, `state.forecast`) — NOT the
  ground-truth `zoneTemp`/`airTemp` directly, to keep the comparison fair (same sensing boundary
  every other mode effectively has, since none of them peek at ground truth either).
- Forecasted ambient temperature and forecasted tariff period (both already computable from
  `ambientNow()` and `tariffPeriodAt()` ahead of the current tick).
- Scheduled events the site already knows about (booked truck deliveries, shift roster) — a real
  facility's AI layer would have this from the same dock-appointment/roster system this sim already
  models, not from magic foresight.
- Recent duty cycle / equipment wear estimate (`estCapMult`, `estUaMult` from the twin estimator).

**Output:** same shape as Modes A/C — `compressorOn[i]` and/or `capacityFraction[i]` — it must drive
the plant through the identical actuator interface, or the comparison in the "Controller comparison"
modal isn't measuring the same thing.

**Not yet built.** When implemented, document its exact control law here in the same format as A/B/C
above, including whatever objective function it optimizes (cost + band-violation penalty, per
TODO.md §5).

---

## Benchmarking harness (§6) — what it measures and how to read it

Every mode above writes into the same live ledger, `state.modeMetrics[cfg.mode]`, updated once per
tick in `tick()` regardless of which mode is active — tagged to whichever mode was actually driving
the plant at that tick. This makes mode-switching mid-run a genuine A/B/C/D log, not four separate
simulations that need reconciling afterward.

**Per mode, tracked continuously:**
| Metric | Field | Meaning |
|---|---|---|
| Time driven | `minutesTracked` | sim-minutes this mode was active |
| Cost by tariff period | `costByPeriod[periodKey].{kwh,cost}` | ₹ and kWh split across offpeak/normal/solar/peak |
| Time-in-band | `minutesInBand` | minutes ALL 3 zones were within `target ± band` of product core temp |
| Excursion severity | `degMinOutside` | Σ (°C beyond band × minutes) — deviation size × duration, across all zones |
| Compressor cycling | `cyclesStarted` | OFF→ON transitions, summed across zones — a wear proxy |
| Shelf-life consumed | `spoilStart` / `spoilLast` | worst-zone spoilage index (0–100) at first vs. most recent tick this mode was active |

**Access points:**
- UI: the "Controller comparison" header badge/modal — live table, one row per mode.
- Programmatic: `window.ColdStorageTwin.getModeMetrics()` — returns a deep-cloned snapshot, safe to
  poll from an external script/notebook for offline analysis.

**How to run a fair benchmark:** hold product, season, and day range fixed; switch `cfg.mode` (via
UI or `ColdStorageTwin.setControllerMode()`) between scenario runs, or run each mode for an equal
number of sim-hours within one session and compare the accumulated rows. Do not compare rows with
very different `minutesTracked` without normalizing (₹/hour, °C·min/hour, cycles/hour) — the modal
already reports the normalized forms (₹/kWh, %-in-band, cycles/hour) for this reason.

**What this benchmark deliberately does NOT yet do** (future work, not in scope for this file):
seeded/reproducible runs (TODO.md §2), so re-running the "same" scenario twice will differ in
exact truck timing and sensor noise draw — fine for a rough comparison, not yet fine for a precise
regression test between mode versions.
