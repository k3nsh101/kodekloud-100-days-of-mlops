### Task

The xFusionCorp Industries MLOps team is standing up the serving path of their `fraud-detector` platform — the route a model takes from a training run to a live prediction endpoint. The backing stack is already running: a SeaweedFS object store (with a seeded training dataset) and an MLflow tracking server. Your task is to drive that path end to end: run the training script to produce a run, complete `register.py` so it registers the run as the `fraud-detector` model and assigns the `production` alias, then start the FastAPI inference server so it serves live predictions from the aliased model.

1. The **MLflow UI** and **SeaweedFS Filer** buttons at the top of the lab open those UIs. Pre-staged state:
   - SeaweedFS bucket `data` holds `transactions.csv`; the `mlflow-artifacts` bucket is empty until a run logs to it.
   - MLflow experiment `fraud-detection` does not exist yet and the Model Registry is empty.
   - Under `/root/code/`: `train.py`, `serve.py`, and `config.yaml` are reference scripts (no edits needed); `register.py` ships with its `register` + promote step as an unfinished `TODO` for you to complete. `train.py` reads the dataset from SeaweedFS and logs the run + artefacts to MLflow; the FastAPI server's background loader polls the registry for `models:/fraud-detector@production` and loads the model once the alias exists.

2. The end state must include:
   - MLflow experiment `fraud-detection` has at least one run; run artefacts are in the `mlflow-artifacts` SeaweedFS bucket.
   - A Registered Model named `fraud-detector` exists with the `production` alias assigned to the version sourced from the run, set in code by `register.py`.
   - `POST http://localhost:8085/predict` with `{"features": [100.5, 12, 3]}` returns `{"prediction": 0}` or `{"prediction": 1}` (tests poll up to 60 s).

This task builds the serving path of the platform: training logs a run + artefacts to MLflow (backed by SeaweedFS object storage), the registry `production` alias is the stable handle production code targets (`models:/fraud-detector@production`), and the serving process resolves that alias at load time — so promoting a new model later is an alias move, not a redeploy.

### Solution

- Update `/root/code/register.py`.

  ```python
  """Register the trained run as `fraud-detector` and promote it.

  train.py logs a run to the `fraud-detection` experiment. This script
  turns that run into the model the serving layer targets: it registers
  `runs:/<run_id>/model` as a version of `fraud-detector`, then puts the
  `production` alias on that version. serve.py resolves
  `models:/fraud-detector@production`, so this alias is the handoff from
  training to serving.

  The run lookup is written for you. Author the TODO, then run:
      python3 /root/code/register.py
  """
  from __future__ import annotations

  import mlflow

  TRACKING_URI = "http://localhost:5000"
  MODEL_NAME = "fraud-detector"
  ALIAS = "production"

  mlflow.set_tracking_uri(TRACKING_URI)
  client = mlflow.tracking.MlflowClient()


  def _latest_run_id() -> str:
      """The most recent run id in the `fraud-detection` experiment."""
      exp = client.get_experiment_by_name("fraud-detection")
      if exp is None:
          raise SystemExit("no `fraud-detection` experiment yet -- run train.py first")
      runs = client.search_runs(
          [exp.experiment_id],
          order_by=["attributes.start_time DESC"],
          max_results=1,
      )
      if not runs:
          raise SystemExit("no runs in `fraud-detection` -- run train.py first")
      return runs[0].info.run_id


  run_id = _latest_run_id()
  print(f"[register] latest run_id={run_id}")

  # TODO: Register runs:/<run_id>/model as a new version of MODEL_NAME,
  # then move the ALIAS (`production`) onto that new version so the
  # serving layer's models:/fraud-detector@production resolves. Use
  # mlflow.register_model(...) and client.set_registered_model_alias(...),
  # and print the version you promoted.
  model_uri = f"runs:/{run_id}/model"

  registered = mlflow.register_model(model_uri=model_uri, name=MODEL_NAME)

  client.set_registered_model_alias(name=MODEL_NAME, alias=ALIAS, version=registered.version)

  print(f"[register] promoted {MODEL_NAME} version {registered.version} to alias {ALIAS}")
  ```

- Run the scripts.

  ```bash
  python3 train.py
  python3 register.py
  python3 serve.py
  ```

- Verify.

  Artefacts are placed in the **SeaweedFS Filer** under `buckets/mlflow-artifacts`.

  From the **MLflow UI**, MLflow experiment fraud-detection has at least one run.

  A Registered Model named fraud-detector exists with the production alias.

  The following returns {"prediction": 0} or {"prediction": 1}.

  ```bash
  curl -X POST http://localhost:8085/predict -H 'Content-Type: application/json' -d '{"features": [100,5,12,3]}'
  ```
