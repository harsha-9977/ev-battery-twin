# EV Battery Digital Twin

A predictive-maintenance prototype that combines simulated battery telemetry, machine-learning models, a REST API and monitoring dashboards.

The project explores **remaining useful life (RUL) prediction and battery failure classification** using Python and XGBoost, with infrastructure for telemetry storage, experiment tracking and monitoring. It also includes a separate interactive dashboard workflow with Random Forest models.

## What the project demonstrates

- Battery-state simulation with state of charge, state of health, voltage, current, temperature and charge cycles.
- Telemetry publishing to PostgreSQL/TimescaleDB, Kafka-compatible Redpanda and MQTT.
- Missing-value handling, outlier filtering, scaling and tabular model development.
- XGBoost regression and classification with model artifacts and evaluation metrics.
- MLflow experiment logging and a live prediction service with Prometheus metrics.
- A Flask REST API and Grafana dashboard configuration.

The telemetry generator is a simulator. This repository does not establish validated performance on a deployed physical battery system.

## Reviewer guide

| Area | Entry point |
| --- | --- |
| Telemetry simulator | [src/simulator/publisher.py](src/simulator/publisher.py) |
| Training with MLflow | [src/models/train.py](src/models/train.py) |
| Training without MLflow | [train_simple.py](train_simple.py) |
| Live inference | [src/inference/live_predictor.py](src/inference/live_predictor.py) |
| Database-backed REST API | [src/api/app.py](src/api/app.py) |
| Infrastructure | [docker-compose.yml](docker-compose.yml) |
| Simulator tests | [tests/test_simulator.py](tests/test_simulator.py) |
| Separate dashboard workflow | [app_advanced.py](app_advanced.py), [train_simple_models.py](train_simple_models.py) |

## Architecture

The simulator writes telemetry to the database and can publish it to Kafka and MQTT. The live predictor reads the latest database telemetry, loads model/scaler artifacts, writes predictions back to the database and exposes Prometheus metrics. The Flask API serves telemetry and prediction records. Grafana configuration supports visualising database data and monitored metrics.

Docker Compose starts the infrastructure services. The Python publisher, predictor and API are started separately on the host; Compose does not automatically launch them.

## Two model workflows

| Workflow | Training data | Outputs | Intended consumer |
| --- | --- | --- | --- |
| XGBoost service pipeline | `datasets/EV_Predictive_Maintenance_Dataset_15min.csv` | `rul_model.pkl`, `failure_model.pkl`, `scaler.pkl` | `src/inference/live_predictor.py` |
| Random Forest dashboard pipeline | `datasets/eviot_dataset.csv` | Target-specific `.joblib` files | Separate dashboard model manager |

These workflows use different schemas and artifacts. Keep them separate rather than treating their model files as interchangeable.

## XGBoost training data

Provide a CSV with the following columns:

| Role | Columns |
| --- | --- |
| Features | `SoC`, `SoH`, `Battery_Voltage`, `Battery_Current`, `Battery_Temperature`, `Charge_Cycles`, `Power_Consumption` |
| Regression target | `RUL` |
| Classification target | `Failure_Probability` |

Use valid binary 0/1 failure labels for the simple training route. The two trainers handle probability-valued labels differently, so inspect that conversion before using another schema. The dataset is not included in the public repository. Its source, licence, units and label definitions should be recorded before reporting a benchmark.

## Run the service workflow locally

Requirements: Python, Docker with Compose, and an appropriate training CSV. A Python 3.10 or 3.11 environment is a reasonable starting point; the complete dependency combination has not been pinned or certified.

```bash
git clone https://github.com/harsha-9977/ev-battery-twin.git
cd ev-battery-twin
python -m venv .venv
# Activate .venv for your operating system.
python -m pip install -r requirements.txt
docker compose up -d
```

Review the Compose ports and local development credentials before starting services. Configure database variables in a local `.env` file as needed: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD` and `DB_NAME`. Keep credentials out of version control.

### 1. Train compatible artifacts

Place the CSV at `datasets/EV_Predictive_Maintenance_Dataset_15min.csv`.

```bash
python train_simple.py
```

This route saves the three `.pkl` artifacts required by the live predictor without requiring MLflow. For experiment logging instead:

```bash
python -m src.models.train
```

The tracked route requires a reachable MLflow server and working artifact storage. The Compose configuration uses MinIO; provision its `mlflow` bucket if it is absent. Dependency compatibility may require adjustment because package versions are open-ended.

### 2. Start the simulator

In a separate activated terminal, from the repository root:

```bash
python -m src.simulator.publisher
```

### 3. Start inference

After the database contains telemetry and the model files exist:

```bash
python -m src.inference.live_predictor
```

### 4. Start the API

In another terminal:

```bash
python -m src.api.app
```

The API defaults to `http://localhost:5001`. Check `http://localhost:5001/health`; it reports database connectivity. These commands describe the code entry points and prerequisites, rather than a claimed end-to-end verified deployment.

## Local service ports

| Service | Default host port |
| --- | --- |
| TimescaleDB | 5432 |
| Redpanda external Kafka listener | 9092 |
| MQTT | 1883 |
| MLflow | 5000 |
| MinIO API / console | 9000 / 9001 |
| Prometheus | 9090 |
| Grafana | 3000 |
| Flask API | 5001 |
| Live predictor metrics | 9100 by default |

Check `config/prometheus.yml` if the predictor runs on the host: Prometheus in Docker must be able to reach the host metrics endpoint. Simulator data is written directly to the database; no separate Kafka-to-database consumer is required by that path.

## API endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/health` | Database connectivity check |
| GET | `/api/battery/latest` | Latest battery telemetry |
| GET | `/api/battery/history` | Historical telemetry |
| GET | `/api/battery/predictions` | Prediction records |
| GET | `/api/battery/stats` | Battery statistics |
| GET | `/api/batteries/list` | Available batteries |
| GET | `/api/alerts` | Battery alerts |

These endpoints belong to `src/api/app.py`. The separate advanced app runs on port 5002 and has a different route set.

## Evaluation and tests

Training code reports RUL regression metrics such as R², RMSE and MAE, and failure classification metrics such as precision, recall, F1 and ROC AUC. Reproduce a run with documented data before quoting results; no fixed performance figures are asserted here.

Simulator tests are available:

```bash
python -m pip install pytest
python -m pytest tests/test_simulator.py -v
```

The tests exercise simulator behaviour and telemetry shape; they do not certify prediction quality or the full infrastructure deployment.

## Current limitations and next improvements

- The training data and trained service artifacts are not distributed with the repository.
- Random train/test splitting may overstate generalisation for correlated battery records. Battery-wise and chronological validation are useful next steps.
- Missing-value filling and outlier thresholds are computed before splitting. Move fitted preprocessing into the training partition for cleaner evaluation.
- The advanced dashboard and the service pipeline have different feature naming conventions and artifact expectations.
- `start_app.ps1` references a root `app.py` that is absent from the current tree. Use the documented service entry point rather than that launcher.
- Monitoring exposes operational metrics; drift detection and automatic model-quality monitoring are not established by those metrics alone.
- The Compose defaults and Flask debug API are intended for local development. Production access control and deployment configuration remain future work.

## Author

[Harsha A P](https://github.com/harsha-9977)
