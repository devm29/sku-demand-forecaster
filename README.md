# Forecast MVP/PoC — SKU × Region weekly forecasting

This repository contains a minimal end‑to‑end MVP that ingests raw CSVs (credit + consumer panel), links them via hashed IDs, computes panel weights, builds weekly features, trains quantile LightGBM models, optionally trains DeepAR/PyMC, ensembles forecasts, and serves results via FastAPI. An example Airflow DAG and Dockerfile are included.

## Directory

```
SKU-Forecasting/
├── README.md
├── requirements.txt
├── docker/
│   └── Dockerfile
├── dags/
│   └── forecast_dag.py
├── src/
│   ├── __init__.py
│   ├── config.py
│   ├── utils/
│   │   ├── hashing.py
│   │   ├── validation.py
│   │   └── metrics.py
│   ├── mlops/
│   │   └── mlflow_utils.py
│   ├── ingest/ingest.py
│   ├── linking/link_panel_credit.py
│   ├── weighting/raking.py
│   ├── features/build_features.py
│   ├── models/
│   │   ├── train_lgb_quantile.py
│   │   ├── train_pymc_hierarchical.py
│   │   └── train_deepar_gluonts.py
│   ├── ensemble/ensemble_and_reconcile.py
│   └── serving/app.py
└── examples/
    └── sample_data/README.md
```

## Run steps — local MVP quickstart

1) Create virtualenv & install deps

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install -r requirements-dev.txt   # optional: pytest, ruff for tests and lint
```

2) Prepare data

- Put sample CSVs in `examples/sample_data/`:
  - `credit.csv` columns: `txn_id, customer_id, txn_date, sku, region, amount, price, quantity`
  - `panel.csv` columns: `panel_id, customer_id, age_group, region, income_bin, household_size`
  - `demographics.csv` (optional)
- Export environment variables (update paths as needed):

```bash
export DATA_DIR=$(pwd)/examples/sample_data
export HASH_SALT="super_secret_salt"
export MLFLOW_TRACKING_URI="http://localhost:5000"   # optional
```

3) Run ingestion

```bash
python -m src.ingest.ingest \
  --credit_csv ${DATA_DIR}/credit.csv \
  --panel_csv ${DATA_DIR}/panel.csv
```

4) Link panel & credit

```bash
python -m src.linking.link_panel_credit
```

5) Compute weights (toy raking example)

```bash
python -m src.weighting.raking
```

6) Build features

```bash
python -m src.features.build_features
```

7) Start MLflow (optional for experiment tracking)

```bash
mlflow server --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./mlruns --host 0.0.0.0 --port 5000
```

8) Train LightGBM quantile models and produce forecasts

```bash
python -m src.models.train_lgb_quantile --quantiles 0.1 0.5 0.9 --horizon 12
```

9) (Optional) Train DeepAR or produce naive fallback forecasts

```bash
python -m src.models.train_deepar_gluonts --horizon 12
```

10) Ensemble & reconcile

```bash
python -m src.ensemble.ensemble_and_reconcile
```

11) Start API (serving)

```bash
uvicorn src.serving.app:app --reload --port 8000
# then POST to /forecast
curl -X POST http://localhost:8000/forecast \
  -H 'Content-Type: application/json' \
  -d '{"sku":"SKU123","region":"NE","horizon_weeks":12}'
```

12) Airflow orchestration (optional)

- Install and initialize Airflow per official docs (Airflow installation is heavyweight; use the official constraints). 
- Copy `dags/forecast_dag.py` into your Airflow DAGs folder, configure connections, and trigger DAG runs.

## Tests and lint

From repo root (with `requirements-dev.txt` installed):

```bash
pytest tests/ -v
make test   # same
make lint   # ruff check and format check
```

## Notes & caveats

- This is an MVP intended for demonstration. Replace toy raking marginals and add stricter validation/monitoring before production.
- DeepAR (GluonTS) and PyMC are optional and can be heavy to install. The DeepAR script falls back to a naive forecast if GluonTS is unavailable.
- Ensure your CSVs have the expected columns and valid datatypes (dates parseable by pandas).

## Minimal sample rows

`examples/sample_data/credit.csv`
```
txn_id,customer_id,txn_date,sku,region,amount,price,quantity
1,ABC,2025-01-02,SKU123,NE,12.5,2.5,5
2,XYZ,2025-01-04,SKU123,NE,7.5,2.5,3
```

`examples/sample_data/panel.csv`
```
panel_id,customer_id,age_group,region,income_bin,household_size
p1,ABC,35-54,NE,50-75k,3
p2,XYZ,35-54,NE,50-75k,2
```

## Docker (optional)

Build & run serving API:

```bash
docker build -t sku-forecasting -f docker/Dockerfile .
docker run --rm -it -p 8000:8000 -e DATA_DIR=/app/examples/sample_data sku-forecasting
```

## What to show in a PoC demo

- Ingestion produces Parquet files in `examples/sample_data/landing/`.
- Features in `examples/sample_data/features/features.parquet`.
- LightGBM forecasts in `examples/sample_data/features/forecast_lgb.parquet`.
- Ensemble forecasts in `examples/sample_data/features/forecast_ensemble.parquet`.
- Query forecasts via the FastAPI endpoint for a SKU/region for N weeks.
