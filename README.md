# cold-storage-simulation

## 📊 Cold Storage Simulation – Factor Coverage Breakdown

| **Factor**                            | **Covered Details**                                                                 | **Skipped/Simplified**                                                                 |
|--------------------------------------|--------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| **Airflow Simulation**               | Zone-based airflow modeling, obstruction effects, multi-chamber ducts               | No 3D CFD yet, no turbulent flow simulation                                             |
| **External Heat Gain**               | Seasonal/day-night profiles, insulation level, wall orientation                     | No real-time weather API yet                                                            |
| **Internal Heat Load (Workers)**     | Worker shift timing → heat injection in zones                                       | Exact paths and density of workers not simulated                                       |
| **Air Curtain / Vestibule**          | Door open-close cycles and insulation impact modeled                                | No fluid simulation of air stream                                                       |
| **Humidity (New)**                   | Configurable RH input, water vapor tracking, condensation on coils                  | No mold/bacterial growth modeling yet                                                   |
| **Stacking Interference**            | Configurable depth, height, packing density → airflow resistance                    | No 3D box collision or micro-zones yet                                                  |
| **Packaging Types**                  | Sack vs crate vs loose → modeled as delay/resistance in airflow and heat exchange   | No material-specific heat retention (e.g. jute vs plastic)                             |
| **Product Thermal Profile**          | Item-level heat capacity, size, mass, and cooling time constant (τ)                 | Surface convection modeling is simplified                                               |
| **Pre-Cooling Delay**               | Exponential decay from initial temp to target temp per product                      | In-crate gradient skipped                                                               |
| **Sensor Bias**                      | Time lag between actual item temp and sensor reading                                | Fixed response model; no noise injection yet                                            |
| **Compressor & Fan Energy**          | Modeled using runtime cycles, load, and zone demands                                | No degradation curve or surge current at startup                                        |
| **Rule-Based Control**               | Traditional threshold-based control strategies implemented                          | No fuzzy or PID controllers yet                                                         |
| **RL-Based Control (Offline/Online)**| Supports online learning via experience buffer + real-time updates                  | Limited to per-zone control; not multivariable predictive yet                          |
| **Multi-Chamber Optimization**       | Cross-chamber load balance, shared compressors, coordinated cooling strategy        | No real-time duct leakage model                                                         |
| **Workload Modeling**                | Dynamic truck arrival & unloading patterns                                          | No forklift path or dock-level thermal model                                            |
| **Shift Modeling**                   | Worker schedules per zone cause time-based internal load                            | No human motion path tracking                                                           |
| **TTI / Spoilage Metrics**           | Zone-based time-temp-integrals with thresholds per product                          | Spoilage = fixed threshold decay; no dynamic microbial modeling                         |
| **Weather Profiles**                 | Time of day and seasonal weather simulate external temperature & RH                 | Not connected to live weather forecast APIs                                             |
| **Visualization**                    | D3.js dashboard, heatmaps, real-time graphs, comparison between controllers         | No 3D warehouse rendering                                                               |
| **Front-End Configurator**           | UI for changing chamber layout, items, packaging, controller type, environment      | Advanced drag-drop layout editing not yet implemented                                  |
| **Simulation Outputs**               | Full per-minute logs, plots, spoilage %, energy by zone/time/controller             | No alerting system or live telemetry                                                    |
                                              |

-----

## 🧪 Setup

### 📁 cold_storage_simulation/

```
cold_storage_simulation/
├── config/                             # All JSON/YAML config files
│   ├── chamber_layouts/                # Chamber shapes, stacking layouts
│   ├── product_profiles/               # Items: mass, size, heat capacity, cooling delay
│   ├── environmental_profiles/         # Seasonal/day-night temperature & humidity
│   ├── workload_profiles/              # Inflow schedules, truck arrival timings
│   ├── packaging_profiles/             # Crates, sacks, loose—thermal resistance etc.
│   ├── chamber_settings.yaml           # Main simulation structure: chambers, zones, airflow config
│   └── simulation_settings.yaml        # Master config: duration, logging, control types

├── simulation_engine/                  # Core simulator logic
│   ├── simulator.py                    # Central time-step engine
│   ├── heat_transfer.py                # Models internal & external heat dynamics
│   ├── airflow_model.py                # Simulates airflow zones, obstruction, duct effect
│   ├── humidity_model.py               # Relative humidity tracking, dehumidification, condensation
│   ├── pre_cooling_model.py            # Models item core-surface lag during cooling
│   ├── spoilage_model.py               # TTI (time-temp-integral) for cold chain risk
│   ├── stacking_model.py               # Models stacking impact on airflow and cooling delay
│   ├── sensor_bias_model.py            # Delays between actual temp and sensor temp
│   ├── workload_model.py               # Simulates dynamic load intake patterns
│   ├── shift_model.py                  # Worker shifts & induced heat gain
│   ├── air_curtain_model.py            # Models insulation effect of air curtains at doors
│   ├── control_orchestrator.py        # Ties control policy to simulation (RL or Rule)
│   └── utils.py                        # Common math, file I/O, interpolation, etc.

├── controllers/                        # Plug-n-play control strategies
│   ├── rule_based_controller.py        # Threshold-based traditional logic
│   ├── rl_policy_controller.py         # Reinforcement learning based agent
│   └── controller_interface.py         # Unified interface for switching controller logic

├── multi_chamber_optimizer/            # Logic to coordinate chambers
│   ├── chamber_coordinator.py          # Load balancing, sync across chambers
│   └── shared_cooling_scheduler.py     # Shared compressor optimization

├── reporting/                          # Analytics, dashboards, and outputs
│   ├── energy_report_generator.py      # Detailed energy usage breakdown (hour/zone)
│   ├── spoilage_report_generator.py    # TTI, degraded product estimation
│   ├── cost_saving_analyzer.py         # Energy difference between controllers
│   ├── logs/                           # Auto-saved simulation logs (CSV/JSON)
│   └── simulation_runs/                # Timestamped run folders

├── visualization/                      # D3.js + static plots
│   ├── dashboards/                     # JS/HTML dashboards (D3.js, heatmaps, zone charts)
│   ├── generate_static_charts.py       # Matplotlib/Plotly fallback
│   └── export_data_for_viz.py          # Formats sim data for front-end charts

├── frontend/                           # Web UI to configure + launch simulations
│   ├── public/                         # Assets, favicon, etc.
│   ├── src/
│   │   ├── components/                 # Config forms, toggles, dashboards
│   │   ├── pages/                      # Home, Results, Config Editors
│   │   └── api/                        # API calls to backend Flask
│   └── index.html

├── api/                                # Flask backend to trigger simulations via frontend
│   ├── endpoints/
│   │   ├── simulate.py                 # POST API to run sim
│   │   └── config_manager.py           # Get/set config via frontend
│   └── app.py

├── notebooks/                          # Experimental analysis & validation
│   ├── validate_airflow_model.ipynb
│   └── verify_rl_vs_rule.ipynb

├── tests/                              # Unit + integration tests
│   ├── test_simulation_engine.py
│   ├── test_controller_interface.py
│   ├── test_energy_report.py
│   └── mocks/                          # Simulated input files for test cases

├── requirements.txt
├── run_simulation.py                   # Entrypoint to run config-based simulation
└── README.md                           # Project overview and setup

```

