# cold-storage-simulation


```
cold_storage_sim/
│
├── main.py
│   # 🚀 Main entrypoint for running the simulation
│
├── requirements.txt
│   # 📦 All dependencies (e.g., numpy, pandas, fastapi, torch, matplotlib)
│
├── config/
│   ├── example_config.json                 # 🔧 Full simulation setup
│   ├── products_catalog.json              # 🍎 Product thermal properties
│   ├── packaging_catalog.json             # 📦 Packaging impact on airflow, thermal
│   ├── airflow_profiles.json              # 💨 Fan specs, layout, airflow logic
│   ├── stacking_profiles.json             # 🧱 Pallet stacking patterns, heights
│   ├── weather_profiles.json              # 🌦️ Time-of-day, seasonal temps
│   └── event_profiles.json                # 🚛 Truck arrivals, work shifts, door events
│
├── simulation_engine/
│   ├── simulation_runner.py               # 🔁 Loads config, runs timestep loop
│   ├── chamber.py                         # 🧊 Chamber configuration & zone layout
│   ├── zone.py                            # 🟦 Individual spatial cell of chamber
│   ├── product.py                         # 🍏 Thermal response of items
│   ├── packaging.py                       # 🧃 Thermal and airflow effect of packaging
│   ├── stacking.py                        # 📦 3D layout of crates/sacks/etc.
│   ├── airflow_model.py                   # 🌬️ Velocity fields per zone
│   ├── heat_gain_model.py                 # 🔥 Wall, door, internal heat sources
│   ├── refrigeration_system.py            # ❄️ Compressor, defrost, energy use
│   ├── environment.py                     # 🌡️ Ambient temps by time-of-day/season
│   ├── event_scheduler.py                 # ⏱️ Door, intake, defrost events
│   ├── energy_tracker.py                  # ⚡ Logs zonewise & periodwise energy
│   ├── sensor_model.py                    # 📡 Simulates crate center vs surface sensors, lag/bias
│   ├── delivery_scheduler.py              # 🚚 Schedules loading/unloading events
│   ├── risk_evaluator.py                  # ⚠️ Computes Time-Temperature Integrals (TTI), risk zones
│   └── multi_chamber_orchestrator.py      # 🧠 Coordinates logic across multiple chambers
│
├── control_logic/
│   ├── base_controller.py                 # 🔌 Shared controller interface
│   ├── rule_based_controller.py           # 📏 Simple thermostat-style logic
│   ├── ai_controller.py                   # 🧠 RL-based learning controller
│   └── reward_functions.py                # 🎯 Temperature deviation + energy tradeoffs
│
├── logging_system/
│   ├── log_manager.py                     # 📝 Manages all logs (temp, events, energy)
│   ├── zone_temp_log.csv
│   ├── energy_log_by_period.csv
│   ├── controller_decision_log.csv
│   ├── tti_risk_log.csv                   # 🆘 Exposure durations & safety thresholds
│   └── summary_report.json                # 📊 Final results consumed by frontend
│
├── reporting/
│   └── report_generator.py                # 📄 Generates charts + insights + recommendations
│
├── utils/
│   ├── config_loader.py                   # 🔧 Validates and parses input configs
│   ├── units.py                           # 📐 Unit conversion helpers
│   ├── interpolation.py                   # 📈 Interpolators (airflow, defrost curves, etc.)
│   └── constants.py                       # 📌 Physical constants used throughout
│
├── visualization/
│   ├── dashboard_server.py                # 🌍 FastAPI backend for dashboard frontend
│   ├── heatmap_generator.py               # 🌡️ Zone temperature / airflow maps
│   ├── comparison_plotter.py              # 📊 Rule vs AI metrics (energy, temp deviation)
│   ├── timeline_visualizer.py             # 📆 Compressor cycles, door open durations, defrost
│   └── export_to_html.py                  # 💾 Generate interactive HTML dashboards
│
├── frontend/
│   ├── public/
│   └── src/
│       ├── App.jsx
│       ├── components/
│       │   ├── ConfigBuilder.jsx          # 📑 Full UI to add/edit chambers, crates, controllers
│       │   ├── SimulationPlayer.jsx       # 🎥 Visual playthrough of simulation
│       │   ├── EnergyBreakdown.jsx        # ⚡ Energy by day/evening/night
│       │   ├── RiskMetricsView.jsx        # 🚨 Product risk / TTI indicators
│       │   └── ComparisonDashboard.jsx    # 📊 Rule vs AI control performance view
│       └── index.html
│
├── tests/
│   ├── test_airflow_model.py
│   ├── test_packaging_resistance.py
│   ├── test_sensor_model.py
│   ├── test_risk_evaluator.py
│   ├── test_delivery_scheduler.py
│   └── test_multi_chamber_orchestrator.py

```
