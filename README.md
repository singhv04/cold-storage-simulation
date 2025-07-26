# 📁 cold_storage_simulation/


# 🧊 Cold Storage Simulation – Factor Coverage Breakdown

| #  | Factor                          | Covered Details                                                                                 | Skipped / Simplified                          |
|----|--------------------------------|--------------------------------------------------------------------------------------------------|------------------------------------------------|
| 1  | **Airflow Modeling**           | Zone-wise airflow, duct/fan placement, velocity, pressure, stacking interference               | CFD-level turbulence models                   |
| 2  | **Stacking Layout**            | Stack height, gaps, blocking airflow, layout logic                                              | Crate deformation dynamics                    |
| 3  | **Product Type**               | Per-product thermal properties (heat capacity, respiration) via config                          | Water content variability, spoilage chemistry |
| 4  | **Packaging Types**            | Crate, box, sack modeling — thermal insulation and airflow resistance                           | Internal packaging layering                   |
| 5  | **Multi-Chamber Support**      | Multiple isolated chambers with independent controls and workloads                              | Air leakage across chambers                   |
| 6  | **Sensor Bias/Delay**          | Lag between surface and core sensor, configurable error margins                                 | Sensor aging/drift over time                  |
| 7  | **Worker Heat Modeling**       | Shift timing, human presence-induced heat, movement-based zone impact                           | Activity-based metabolic rate variation       |
| 8  | **External Weather Influence** | Hourly/daylight profile, seasonal variation, wall transfer rate, wall orientation               | Real building shape and solar load            |
| 9  | **Refrigeration Load**         | Fan, compressor power, duty cycle, energy tracking, peak loads                                  | Refrigerant gas type behavior                 |
| 10 | **Control Logic (Rule-based)** | On/Off thermostat thresholds, hysteresis logic                                                  | Adaptive logic for subzones                   |
| 11 | **AI Control Logic**           | RL/ML agent using zone temps, energy, risk for decision-making                                  | Real-time re-training                         |
| 12 | **Energy Consumption Tracking**| Compressor, fan breakdowns by time window (day/night), total energy used                        | Real electrical losses                        |
| 13 | **Cold Chain Risk (TTI)**      | Time-Temperature Integral model to quantify spoilage or risk                                    | Ethylene-based deterioration                  |
| 14 | **Door Open Events**           | Scheduled door open/close profiles, airflow loss, vestibule modeling                            | Door surface shape effects                    |
| 15 | **Air Curtain Simulation**     | Configurable insulation factor during door open events                                          | CFD of barrier mixing                         |
| 16 | **Product Loading/Unloading**  | Truck arrival events, shift-based intake/output heat profiles                                   | Product pre-cooling delay                     |
| 17 | **Chamber Zones**              | Divided spatial zones for temperature and airflow tracking                                      | 3D voxel-level heat modeling                  |
| 18 | **Weather Profiles**           | Monthly/daily/hourly profiles via JSON, affects wall heat transfer                              | Humidity/latent heat integration              |
| 19 | **Visualization (Frontend)**   | D3.js charts, 3D airflow + zone heatmap, config dashboards                                      | Fluid dynamics particles                      |
| 20 | **Custom Configurations**      | Full flexibility in input: items, stacking, air layout, walls, sensors, workers                 | Real-time drag-drop stack planner             |
| 21 | **Fault Simulation (optional)**| Placeholder for later: cooling failure, sensor loss, delayed loading                            | Currently unimplemented                       |
| 22 | **Insulation Materials**       | Wall insulation effect, vestibule resistance, crate resistance                                  | Degradation over years                        |
| 23 | **Thermal Inertia**            | Lag in crate center vs surface, per-product delay configuration                                 | No heat diffusion equation                    |
| 24 | **Energy Cost Optimization**   | Total consumption for AI vs Rule, zone-by-zone efficiency                                       | Real billing tariffs                          |
| 25 | **Data Logging & Reporting**   | Hourly logs, run comparisons, PDF/CSV reports with insights                                     | None                                           |
| 26 | **Testing & Validation**       | Modular test suite per component                                                                | Physical validation vs real warehouse         |


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



```
├── config/                            # 🔧 All configurable parameters
│   ├── example_config.json            # Master configuration file
│   ├── weather_profiles.json          # Hourly/monthly weather data
│   ├── products_catalog.json          # Vegetable/fruit properties
│   ├── packaging_catalog.json         # Crates, sacks, boxes materials
│   ├── stacking_profiles.json         # Stack layout, height, airflow resistance
│   ├── airflow_profiles.json          # Fan placement, ducting, circulation zones
│   ├── event_profiles.json            # Loading/unloading/truck schedules
│   ├── worker_shifts.json             # Heat and events induced by workers
│   ├── sensor_profiles.json           # Bias, lag, accuracy per sensor type
│   └── insulation_features.json       # Air curtains, vestibule settings

├── simulation_engine/                 # 🧠 Physics-based simulation engine
│   ├── simulation_runner.py           # Main orchestrator
│   ├── chamber.py                     # Models the cold chamber as thermal system
│   ├── zone.py                        # Partitioned zones inside chamber
│   ├── airflow_model.py               # Simulates air velocity, pressure
│   ├── heat_gain_model.py             # Heat ingress from walls, doors, humans
│   ├── refrigeration_system.py        # Compressor, fan energy + cooling logic
│   ├── stacking_model.py              # Models airflow resistance due to layout
│   ├── product.py                     # Perishable item thermal profiles
│   ├── packaging.py                   # Impact of crates, sacks, wrap types
│   ├── weather_engine.py              # External weather influence model
│   ├── sensor_model.py                # Lag, inaccuracy modeling
│   ├── event_scheduler.py             # Handles doors open, deliveries, workload
│   ├── delivery_scheduler.py          # Manages truck schedules
│   ├── energy_tracker.py              # Logs compressor/fan energy use
│   ├── worker_shift_model.py          # Adds heat/events from worker activity
│   ├── vestibule_model.py             # Simulates vestibule / air curtain effect
│   ├── fault_model.py                 # Optional failure simulation (future)
│   └── chamber_manager.py             # For multiple chambers coordination

├── control_logic/                     # 🤖 Rule-based vs AI controllers
│   ├── rule_based_controller.py       # Threshold-based traditional logic
│   ├── ai_controller.py               # RL-based dynamic policy controller
│   ├── control_utils.py               # Shared methods and state interface
│   └── multi_chamber_orchestrator.py # Controls multi-chamber interactions

├── reporting/                         # 📊 Risk, performance, and summary reports
│   ├── report_generator.py            # Generates PDF/HTML reports
│   ├── comparison_plotter.py          # Plots AI vs Rule metrics
│   └── risk_evaluator.py              # TTI, risk of spoilage, etc.

├── utils/                             # 🧰 Utilities
│   ├── config_loader.py               # Loads and validates config
│   ├── unit_converter.py              # Handles Celsius↔Fahrenheit, kWh↔Joules
│   ├── logger.py                      # Logging formatter
│   └── validation.py                  # Schema checks for config files

├── data/                              # 📁 Raw and synthetic input data
│   ├── weather_data/                  # Real weather CSV files
│   └── product_samples/               # Example product thermal logs

├── logs/                              # 📁 Auto-generated logs
│   ├── temp_logs/                     # Temperature history per zone
│   ├── energy_logs/                   # Compressor/fan kWh use
│   └── events_log/                    # Door open, staff presence, etc.

├── simulation_runs/                   # 📁 Saved outputs from previous runs
│   ├── run_YYYYMMDD_HHMM/
│   │   ├── config_used.json
│   │   ├── temp_log.csv
│   │   ├── energy_log.csv
│   │   ├── tti_risk.json
│   │   └── final_report.pdf

├── api/                               # 🌐 Backend API (FastAPI or Flask)
│   ├── main.py                        # Entrypoint for simulation API
│   ├── endpoints/
│   │   ├── simulation.py              # Start/stop simulation
│   │   ├── config.py                  # Upload/download configs
│   │   └── metrics.py                 # Serve real-time stats to frontend
│   ├── scheduler.py                   # Runs queued simulations
│   └── websocket.py                   # Push updates (Live temp, energy, etc.)

├── frontend/                          # 🖥️ Web app with D3 visualizations
│   ├── public/
│   │   └── index.html
│   ├── src/
│   │   ├── App.jsx
│   │   ├── pages/
│   │   │   ├── Configurator.jsx       # Upload/customize simulation config
│   │   │   └── Compare.jsx            # Compare AI vs Rule-based results
│   │   ├── components/
│   │   │   ├── Dashboard.jsx          # Real-time metrics
│   │   │   ├── Charts.jsx             # 📊 D3.js line/bar/heatmap charts
│   │   │   ├── ZoneHeatmap3D.jsx      # 3D airflow/temp visualizer
│   │   │   ├── ConfigEditor.jsx       # UI for creating/editing configs
│   │   │   └── EnergyBreakdownCard.jsx# Day/Night/Afternoon energy use
│   │   ├── utils/api.js               # Axios/API bridge to backend
│   │   └── styles/                    # Tailwind/custom styles
│   ├── vite.config.js
│   └── tailwind.config.js

├── notebooks/                         # 📒 Analysis & prototyping
│   ├── airflow_vs_stacking.ipynb
│   └── ai_vs_rule_summary.ipynb

├── tests/                             # 🧪 All unit and integration tests
│   ├── simulation_engine/
│   │   ├── test_chamber.py
│   │   ├── test_airflow_model.py
│   │   ├── test_sensor_model.py
│   │   └── test_energy_tracker.py
│   ├── control_logic/
│   │   ├── test_rule_based_controller.py
│   │   └── test_ai_controller.py
│   ├── utils/
│   │   ├── test_config_loader.py
│   │   └── test_validation.py
│   ├── api/
│   │   └── test_simulation_routes.py
│   ├── reporting/
│   │   ├── test_report_generator.py
│   │   └── test_risk_evaluator.py
│   ├── fixtures/
│   │   ├── sample_config.json
│   │   └── mock_weather.csv
│   └── conftest.py

├── .github/
│   └── workflows/
│       └── ci.yml                     # CI pipeline: pytest + lint + build

├── main.py                            # 🧵 CLI interface to run simulations
├── README.md                          # 📘 Project overview, setup, usage
├── requirements.txt                   # 🐍 Python dependencies
├── package.json                       # 📦 Frontend dependencies
└── .env                               # 🔐 Secrets (weather API keys, etc.)
```

