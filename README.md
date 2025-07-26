# 📁 cold_storage_simulation/


# 🧊 Cold Storage Simulation – Factor Coverage Breakdown

| **Factor**                         | **Covered Details**                                                                                                                                          | **Skipped/Simplified**                                                                                      |
|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| **Airflow Simulation**             | Directional zone-level airflow with adjustable fan positions, speeds, and stacking influence. Air curtain effect considered.                                 | No 3D voxel-based CFD; airflow is simplified to zone-level with influence maps.                             |
| **Stacking & Blocking**            | Product stacking height, crate layout, and airflow interference modeled via airflow-weighting logic.                                                         | Detailed object collision or full 3D layout physics not included.                                           |
| **Heat Gain – External**           | Based on wall/roof orientation, insulation material, and weather profile (day/night/season).                                                                  | Solar radiation angles not dynamically calculated.                                                          |
| **Heat Gain – Internal (Workers)** | Worker heat modeled by shifts, presence, metabolic rate, and zone impact.                                                                                     | Movement paths and human distribution inside room not spatially resolved.                                   |
| **Heat Gain – Door Events**        | Door opening frequency, duration, and insulation via air curtain considered.                                                                                 | Turbulence or pressure exchange not modeled in full physical accuracy.                                      |
| **Insulation Configs**             | Floor, wall, ceiling, and door insulation R-values are configurable and per zone.                                                                             | Aging/degradation of insulation over time not modeled.                                                      |
| **Product Properties**             | Each product has thermal mass, specific heat, moisture level, spoilage limits, and storage profile.                                                          | No microbial modeling or enzymatic kinetics (e.g., ethylene effect).                                        |
| **Packaging Types**                | Crates, sacks, bundles modeled with size, material, and insulation resistance.                                                                                | Complex irregular shapes are abstracted as bounding box containers.                                         |
| **Product Pre-cooling Delay**      | Surface vs core temp lag is modeled; items cool slower from center with time constants based on packaging and size.                                          | No thermal gradient within object (not voxelized).                                                          |
| **Compressor/Fan Control**         | Rule-based threshold logic and AI optimization controller (with RL model) available.                                                                          | AI currently simulated as policy lookup, not online training.                                               |
| **Energy Consumption**             | Compressor/fan/lighting energy tracked by operational time and power draw. Aggregated per hour & per condition.                                               | Startup surge, degradation, and real compressor models simplified to runtime power draw.                    |
| **Cold Chain Risk (TTI)**          | Time-Temperature Integral calculated per product per zone; spoilage estimations generated.                                                                   | No product-specific spoilage chemistry; generalized TTI logic applied.                                      |
| **Weather & Time of Day**          | Hourly profiles for temperature (day/night/evening) and seasonal months.                                                                                     | No humidity or wind direction effects modeled yet.                                                          |
| **Multi-Chamber Coordination**     | Chambers simulated independently or in coordinated way; inter-chamber logic supported for shared load.                                                       | Interconnected ducts or cross-flow leakage not yet simulated.                                               |
| **Sensor Bias**                    | Configurable temperature lag between crate surface and core sensors; sensor position bias can be simulated.                                                  | No sensor error noise or drift modeling (only delay).                                                       |
| **AI Controller Optimization**     | RL-based controller attempts to optimize cooling effort for energy saving while meeting thresholds and minimizing spoilage.                                 | Offline trained policy only; no adaptive online RL implemented in real-time loop.                           |
| **Worker Shifts & Presence**       | Models different operational loads based on staff schedule (e.g., loading/unloading in morning/evening).                                                    | Does not track specific human paths or actions — only zone-wise presence.                                   |
| **Air Curtains / Vestibules**      | Door models include presence or absence of air curtain; impact on airflow and heat gain is modeled using insulation factor.                                 | No dynamic airflow turbulence simulation; simplified door resistance.                                       |
| **Pre/Post Run Reporting**         | Detailed logging of energy use, spoilage, TTI, zone temps, and controller actions; reports in CSV, JSON, and visual plots.                                  | Reports do not include confidence bounds or predictive accuracy ranges.                                     |
| **Interactive Visualization**      | Heatmaps, airflow direction, temperature curves, zone-wise risk using D3.js and matplotlib.                                                                  | No 3D voxel rendering or real physics engine rendering (e.g., Unity/Cesium).                                |
| **Frontend Config Editor**         | Users can configure all parameters: chamber setup, products, airflow, weather profile, insulation, controllers, etc.                                         | No real-time editing during simulation run (for now); editing is pre-run only.                              |
| **Simulation Runtime Controls**    | Run simulations for N hours, simulate fast-forward or step-by-step, compare multiple strategies.                                                             | No distributed multi-core parallel runs yet.                                                                |
| **Scenario Comparison Dashboard**  | Rule vs AI control compared side-by-side by energy usage, spoilage, and performance metrics with visual plots.                                               | No predictive extrapolation or simulation forecasting yet.                                                  |

---

# 🔧 Realism Score Summary

- 🔵 **Very Close to Real-World (19/26)** — strong approximations or configurable proxies.
- 🟡 **Close but Simplified (5/26)** — not fully physics-based (e.g., CFD, gas chemistry).
- 🔴 **Skipped for Now (2/26)** — failure injection, advanced diffusion.

---

# 🧪 Next Steps to Improve Realism

1. CFD-based airflow heat transfer
2. Real data from working cold storages for validation
3. Spoilage chemical modeling
4. Humidity & condensation control (latent heat)



# 🧪 Setup

```
cold_storage_simulation/
├── api/                               # Flask/FastAPI backend to control simulation via UI or APIs
│   ├── __init__.py
│   ├── endpoints/
│   │   ├── simulation.py              # Run simulation, get results
│   │   ├── config_loader.py           # Update/load configs via API
│   │   └── control_toggle.py          # Switch between rule-based and AI
│   └── schemas/
│       ├── product_schema.py
│       └── simulation_request.py

├── configs/                           # Modular config files for each factor and environment
│   ├── products/                      # Product-specific settings (cp, moisture, etc.)
│   │   ├── mango.json
│   │   └── potato.json
│   ├── packaging/                     # Crate/sack/bag metadata (thermal resistance, size)
│   │   ├── plastic_crate.json
│   │   ├── jute_sack.json
│   │   └── tied_bundle.json
│   ├── chambers/                      # Chamber layout & zone settings
│   │   ├── single_chamber.json
│   │   └── multi_chamber.json
│   ├── insulation.json                # Walls, doors, roof, floor insulation factors
│   ├── weather_profile.json           # Time-of-day and seasonal temperature
│   ├── airflow.json                   # Fan positions, speeds, zone airflow configs
│   ├── worker_shifts.json             # Worker presence heat model
│   ├── door_schedule.json             # When doors open/close + air curtain data
│   ├── controller_settings.json       # Rule and AI controller configs
│   └── simulation_settings.json       # Duration, time_step, visualization flags, etc.

├── core/                              # Core simulation models (physical & decision logic)
│   ├── thermal/
│   │   ├── temperature_solver.py      # Computes zone/product temperature updates
│   │   ├── product_cooling.py         # Handles pre-cooling delays (surface vs core temp)
│   │   └── heat_transfer_utils.py     # Conduction/convection computations
│   ├── airflow/
│   │   ├── airflow_simulator.py       # Weighted airflow path solver across zones
│   │   └── fan_placement_optimizer.py # Advanced airflow balancing (optional)
│   ├── control/
│   │   ├── rule_based.py              # Traditional threshold-based control
│   │   ├── ai_controller.py           # RL or ML model-based intelligent agent
│   │   └── controller_interface.py    # Wrapper interface for switching
│   ├── risk/
│   │   ├── tti_calculator.py          # Time-temperature-integral logic
│   │   └── product_spoilage_estimator.py
│   ├── energy/
│   │   ├── energy_tracker.py          # Track compressor/fan energy by component
│   │   └── energy_optimizer.py        # Reduce power with AI trade-offs
│   └── heat_gain/
│       ├── external_weather_gain.py
│       ├── door_opening_loss.py
│       ├── worker_shift_gain.py
│       └── air_curtain_effect.py

├── engine/                            # Simulation runner and orchestrator
│   ├── initializer.py                 # Load settings, validate configs, initialize state
│   ├── simulation_runner.py           # Core engine that loops over timesteps
│   ├── logger.py                      # Logs all results and summaries
│   └── engine_config.py               # Abstract engine setup to allow plug-and-play modules

├── frontend/                          # Full UI (Flask+HTML+D3 or React)
│   ├── templates/
│   │   └── index.html                 # Main dashboard
│   ├── static/
│   │   ├── js/
│   │   │   ├── d3_visualizer.js       # Heatmaps, airflow, live plots
│   │   │   └── config_form.js         # Interactive config selector
│   │   └── css/
│   │       └── style.css
│   └── app.py                         # Flask app entry point for UI

├── logs/                              # Raw simulation logs and debugging
│   ├── temp_logs.csv
│   ├── energy_logs.csv
│   ├── risk_logs.csv
│   └── controller_actions.csv

├── reports/                           # Final result reports
│   ├── energy_report.pdf              # Full energy and savings breakdown
│   ├── spoilage_report.pdf            # Cold chain risk summary
│   └── summary.json

├── simulation_runs/                   # Folder to store run outputs
│   ├── run_2025_07_26_10AM/
│   │   ├── config_snapshot.json
│   │   ├── logs/
│   │   ├── plots/
│   │   └── results.json
│   └── ...

├── tests/                             # Unit and integration tests
│   ├── test_airflow_simulator.py
│   ├── test_thermal_model.py
│   ├── test_ai_controller.py
│   ├── test_config_loading.py
│   └── test_engine_loop.py

├── utils/                             # Utility functions & helpers
│   ├── json_loader.py
│   ├── time_utils.py
│   ├── unit_converter.py
│   └── logger.py

├── notebooks/                         # Jupyter notebooks for experiments and debugging
│   ├── ai_controller_training.ipynb
│   ├── airflow_validation.ipynb
│   └── pre_cooling_simulation.ipynb

├── visualization/                     # Static & interactive plots, exports
│   ├── energy_plot.py
│   ├── temperature_map.py
│   ├── spoilage_chart.py
│   └── generate_html_dashboard.py

├── requirements.txt                   # All required dependencies
├── run_simulation.py                  # CLI entry point to launch simulation
└── README.md                          # Full overview and how-to-run guide

```

