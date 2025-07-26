# 📁 cold_storage_simulation/

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

