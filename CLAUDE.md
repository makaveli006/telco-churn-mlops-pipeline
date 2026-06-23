# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## GitHub Rules

For branching model and commit message conventions, see `C:\Users\subin\OneDrive\Desktop\Hiring-Espresso\.claude\rules\github-rules`

## Commands

### Local Environment Setup (Python venv)
```bash
pyenv local 3.11.9
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

### Training Pipeline
```bash
python scripts/run_pipeline.py --input data/raw WA_Fn-UseC_-Telco-Customer-Churn.csv --target Churn

# Optional overrides
python scripts/run_pipeline.py --input data/raw WA_Fn-UseC_-Telco-Customer-Churn.csv --target Churn --threshold 0.35 --test_size 0.2 --experiment "Telco Churn"
```

### Testing
```bash
python scripts/test_pipeline_phase1_data_features.py   # data processing + feature engineering
python scripts/test_pipeline_phase2_modeling.py        # model training + evaluation
python scripts/test_fastapi.py                         # FastAPI endpoints
```

### Local Serving
```bash
python -m uvicorn src.app.main:app --host 0.0.0.0 --port 8000
```

### MLflow UI
#### Without it, MLflow UI defaults to looking at ./mlruns anyway — but specifying it explicitly ensures it reads from the right place, especially if you're running the command from the project root where your mlruns/ lives.
```bash
mlflow ui --backend-store-uri file:./mlruns
```

### Docker
```bash
# Using docker-compose.yml (recommended)
docker compose up -d --build   # build + run
docker compose down            # stop + remove

# Using dockerfile directly
docker build -t telco-churn-app .
docker rm -f telco-churn
docker run -d -p 8000:8000 --name telco-churn telco-churn-app

# Health check
curl http://localhost:8000/
```

## Architecture

### Pipeline Flow

**Training** (`scripts/run_pipeline.py`):
Raw CSV → Great Expectations validation → `preprocess_data()` → `build_features()` → XGBoost training → MLflow logging

**Serving** (`src/app/main.py` + `src/serving/inference.py`):
`POST /predict` or Gradio `/ui` → feature transformation → MLflow pyfunc model → prediction string

### Feature Engineering Consistency

Training and serving must apply identical transformations — this is the most critical coupling point:

- **Training** (`src/features/build_features.py`): binary Yes/No + Male/Female → 0/1; multi-category → `pd.get_dummies(drop_first=True)`; booleans → int
- **Serving** (`src/serving/inference.py`): uses fixed `BINARY_MAP` dict; same `get_dummies` parameters; aligns columns to `feature_columns.txt` from training artifacts

### MLflow Artifacts

- Tracking URI: `file://{project_root}/mlruns` (file-based, no server)
- Experiment name: `"Telco Churn"` (default)
- Logged artifacts per run: `model/`, `feature_columns.txt`, `preprocessing.pkl`
- Tracked metrics: `precision`, `recall`, `f1`, `roc_auc`, `train_time`, `pred_time`, `data_quality_pass`

### Docker / Container

- Model artifacts are baked into the image at build time from a specific MLflow run ID (currently `3b1a41221fc44548aed629fa42b762e0` hardcoded in `dockerfile:23-25`)
- `PYTHONPATH=/app/src` — required for `from serving.inference import ...` to resolve without `src.` prefix
- uvicorn entrypoint: `src.app.main:app`

### CI/CD

- Trigger: push to `main` only
- Action: builds Docker image → pushes to Docker Hub as `anasriad8/telco-fastapi:latest`
- Required secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`
- ECS deployment (AWS Fargate + ALB) is triggered manually after image push

### API Endpoints

- `GET /` — health check (required for ALB health checks)
- `POST /predict` — accepts `CustomerData` Pydantic model (18 fields); returns `{"prediction": "Likely to churn"}` or `{"prediction": "Not likely to churn"}`
- `GET /ui` — Gradio interface mounted via `gr.mount_gradio_app()`

### Key Defaults

| Parameter | Default | Notes |
|-----------|---------|-------|
| `threshold` | `0.35` | Classification threshold (lower = higher recall) |
| `test_size` | `0.2` | Train/test split |
| `scale_pos_weight` | dynamic | Computed from class ratio to handle imbalance |
| XGBoost `n_estimators` | `301` | Tuned |
| XGBoost `learning_rate` | `0.034` | Tuned |
| XGBoost `max_depth` | `7` | Tuned |

### Local vs Container Model Paths

- **Local dev**: inference loads from `./mlruns/.../artifacts/model` or `src/serving/model/<run_id>/`
- **Container**: inference loads from `/app/model` (copied from specific run at build time)
