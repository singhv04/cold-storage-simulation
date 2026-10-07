# Cold Storage Simulation

A self-contained, browser-based simulation of a 3-chamber Indian cold storage facility — physics-driven temperature/humidity/spoilage dynamics, a closed-loop "digital twin" estimator layer, realistic operational scheduling (deliveries, shifts, maintenance, put-away/outbound logistics), and a live control-room dashboard.

**Live demo:** [singhv04.github.io/cold-storage-simulation](https://singhv04.github.io/cold-storage-simulation/)
**Source:** [`simulation/cold_storage_simulation.html`](simulation/cold_storage_simulation.html) — a single HTML file, no build step, no dependencies.

## What it is

This simulates one facility with 3 chambers, **each holding its own product at its own setpoint** (e.g. potato in Bulk, frozen meat in Deep-freeze), at a **selectable location** whose state sets both the tariff and the climate, — by default a commercial-scale store (~2,650 t) whose refrigeration plant is sized for the selected product; the original small demo rooms (~35 t) are still selectable:

- **Zone A — Receiving**: the only chamber with a dock door; inbound trucks arrive here on a booked schedule tied to the selected product's real delivery cadence.
- **Zone B — Bulk racks**: general chilled/produce storage.
- **Zone C — Deep freeze**: frozen storage; put-away automatically routes frozen-appropriate items (≤ −10°C) here, everything else to Zone B.

Each chamber is modeled as three coupled thermal bodies — **wall, air, and product** — each with its own thermal mass, so a warm pallet doesn't instantly match the air around it, and outdoor temperature swings lag through the insulated wall before reaching the air. Compressor cooling acts on the air; spoilage tracks the *product's* core temperature, which is what actually matters for food safety.

On top of the physics sits a **closed-loop digital twin**: the "smart" layer never sees the true simulated temperature directly, only a noisy, occasionally-dropped sensor feed. It maintains its own estimate of compressor capacity and wall insulation, corrects that estimate against the sensor feed every tick, flags anomalies (equipment wear vs. sensor drift) once residual error won't shrink, forecasts 3 hours ahead from its own calibrated model, and exposes everything through a `window.ColdStorageTwin` JS API for an external AI/automation layer to read state and issue commands.

## Real-world grounding

- **9 product profiles**, each with real specific heat (derived from water content via Siebel's equation), realistic storage targets, and shelf-life kinetics — including 5 Indian items (potato, onion, tomato, banana, mango) with correct chilling-injury-aware targets (tomato/banana/mango are deliberately held at 12–14°C, not near 0°C).
- **5 controller strategies**, run side by side on identical conditions: Two-position on/off (the India industry default), Adaptive band, Variable-speed PI (PID/VFD), **Predictive optimizer (MPC)** — a 12-hour economic model-predictive controller optimizing energy, maximum-demand charges and restart losses against keeping product in its safe band — and **AI (self-learning)**, the same planner plus online learning of the building's real heat loads, delivery and put-away patterns, weather and per-chamber safety margins. The simulated building deliberately differs from its spec sheet (hidden per-run deviations) so there is something real to learn; an AI learning panel shows learned vs true values. A live scoreboard shows each controller's % of time in the safe range and total bill.
- **Real plant layout**: each commercial chamber has a rack of 2-4 compressors staged lead-lag (fixed-speed step-thermostat racks for the relay modes, a variable-speed lead plus fixed lags for PID/AI), with real anti-short-cycle limits, star-delta/DOL/soft starters, evaporator fan heat, and a PI loop commissioned by lambda tuning.
- **Real-world physics checked line by line** (see CONTROL_IO.md "Realism audit"): lift-based COP with a head-pressure floor, coil output that falls as room air nears coil temperature, defrost method chosen by coil temperature (none / off-cycle / electric), roof solar gain, produce transpiration keeping rooms at 80-90% RH, door openings and lighting in every chamber, controllers acting on real sensors with offsets and lag, and hidden as-built deviations from the spec sheet.
- **India-tuned ambient model**: 4 fixed seasons or a real calendar (monthly climate normals, seasons changing through the run), asymmetric diurnal temperature curve, and 14 selectable state-DISCOM tariffs, each with time-of-day energy rates and a maximum-demand (₹/kVA·month) charge.
- **Realistic scheduling, not random events**: dock appointments booked from the product's real delivery interval, two proper worker shifts with headcount-driven heat load, maintenance dispatched with a realistic lead time (not instant), forklift put-away/outbound trips with real travel time between chambers, and gradual (not sudden) equipment wear.

## Using it

Open `simulation/cold_storage_simulation.html` in any browser — no server or build step needed. Three header controls open deeper views:

- **Events** — a day-by-day, filterable log of every delivery, put-away, outbound shipment, defrost cycle, maintenance visit, shift change, and alert, with real start times and measured durations.
- **Glossary** — a searchable table of every technical term used on the page, in plain language, with an example from the running simulation, the equation behind it, and how it differs from real life.
- **Simulated twin, not live** — the honest design-summary: what's physically modeled, what's still simplified, and what it would take to connect this to a real facility.

## Known limitations

This is a simulation, not a connected system — there is no real hardware behind it. Each chamber is modelled as one well-mixed room (no stratification or hot spots). See the in-app design summary and CONTROL_IO.md "Realism audit" for the full list of what is still simplified.
