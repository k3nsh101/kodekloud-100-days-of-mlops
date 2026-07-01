### Task

The xFusionCorp Industries ML platform team ships a full fraud-detection training pipeline—data validation, Optuna tuning across two model families, model selection against a release threshold, Model Registry registration with a release-lane alias, and a consolidated training report—all wired together behind a single `make train-pipeline` command. The pre-staged system does not currently run end-to-end: each `make train-pipeline` invocation surfaces a wiring issue, and the integration across the `Makefile`, `src/select_model.py`, and `src/register.py` needs attention before the release checklist passes. Your task is to correct the wiring so `make train-pipeline` runs cleanly end-to-end, the MLflow Model Registry holds a `fraud-detector` version under the `staging` alias, and `reports/training_report.json` aggregates every upstream artefact.

1. The MLflow tracking server is already running on port `5000`. The **MLflow UI** button at the top of the lab can be opened to confirm—the dashboard loads with an empty `fraud-detection-tuning` experiment.

2. The project layout under `/root/code/fraud-detection/`:
   - `data/train.csv` – The 200-row synthetic binary-classification dataset the rest of the Training section uses.
   - `src/validate_data.py` – Schema + null-check gate. Writes `reports/validation_status.json`. Correct.
   - `src/tune.py` – Runs 10 Optuna trials across RandomForest and GradientBoosting, each logged as an MLflow run tagged with `model_type` + `params.{n_estimators,max_depth}` + `metrics.f1_score` + the fitted model artefact. Correct.
   - `src/select_model.py` – Picks the winning run by the training metric and writes `reports/selection.json`. Needs attention.
   - `src/register.py` – Registers the selected run's model as `fraud-detector` and assigns the release-lane alias. Needs attention.
   - `src/report.py` – Aggregates every upstream artefact into `reports/training_report.json`. Correct.
   - `Makefile` – `train-pipeline` target runs the five stages in order. Needs attention.

3. Run `make train-pipeline` from `/root/code/fraud-detection/` to surface each issue in turn. Open the offending file in the VS Code editor, correct the wiring, and re-run until the pipeline completes without non-zero exit.

4. The end state must include:
   - `make train-pipeline` completes without non-zero exit.
   - The `fraud-detection-tuning` MLflow experiment carries at least five trial runs, each with `metrics.f1_score`.
   - `reports/selection.json`, `reports/validation_status.json`, and `reports/training_report.json` are all present. The training report carries `best_model`, `best_params`, `metrics`, `total_trials`, and `validation_status` keys; `validation_status` is `"ok"` and `total_trials` is an integer ≥ 5.
   - The MLflow Model Registry (**MLflow UI → Models**) shows a `fraud-detector` registered model with at least one version. That version carries the `staging` alias and no `production` alias.

Run `make train-pipeline` once against the scaffold as-is; the first wiring issue surfaces immediately. Each subsequent re-run reveals the next stage's problem. Every fix is a one-line edit in one of the three files listed above.

### Solution

- Change the directory

  ```bash
  cd fraud-detection
  ```

- Update the `src/select_model.py`

  Change `accuracy` to `f1_score`

  ```python
  """Stage 3 — Model selection.

  Reads every run in the `fraud-detection-tuning` experiment, picks
  the best candidate by the training metric, validates it against the
  release threshold, and persists the selection to
  `reports/selection.json` for the register stage.
  """
  import json
  import os
  import sys

  import mlflow

  TRACKING_URI = "http://localhost:5000"
  EXPERIMENT = "fraud-detection-tuning"
  REPORTS_DIR = "/root/code/fraud-detection/reports"
  SELECTION_JSON = os.path.join(REPORTS_DIR, "selection.json")

  RELEASE_THRESHOLD = 0.4


  def main():
      mlflow.set_tracking_uri(TRACKING_URI)
      client = mlflow.MlflowClient()
      exp = client.get_experiment_by_name(EXPERIMENT)
      if exp is None:
          sys.exit(f"[select] experiment {EXPERIMENT!r} not found.")

      runs = mlflow.search_runs(
          experiment_ids=[exp.experiment_id],
          order_by=["metrics.f1_score DESC"],
          max_results=200,
      )
      if runs.empty:
          sys.exit(
              f"[select] no runs in experiment {EXPERIMENT!r} — the tune "
              "stage has not produced any candidates yet."
          )

      best = runs.iloc[0]
      score = float(best["metrics.f1_score"])
      if score < RELEASE_THRESHOLD:
          sys.exit(
              f"[select] best candidate ({score:.4f}) is below the "
              f"release threshold ({RELEASE_THRESHOLD})."
          )

      selection = {
          "run_id": best["run_id"],
          "model_type": best.get("tags.model_type", ""),
          "f1_score": score,
      }
      os.makedirs(REPORTS_DIR, exist_ok=True)
      with open(SELECTION_JSON, "w") as f:
          json.dump(selection, f, indent=2)
      print(f"[select] {selection}")


  if __name__ == "__main__":
      main()
  ```

- Update `src/register.py`

  Change `RELEASE_ALIAS` to `staging`

  ```python
  """Stage 4 — Register the selected model.

  Reads the selection written by the previous stage, registers the
  selected run's model as `fraud-detector` in the MLflow Model
  Registry, and assigns the release-lane alias so the serving layer
  can fetch the right version by name.
  """
  import json
  import os
  import sys

  import mlflow
  from mlflow.tracking import MlflowClient

  TRACKING_URI = "http://localhost:5000"
  REPORTS_DIR = "/root/code/fraud-detection/reports"
  SELECTION_JSON = os.path.join(REPORTS_DIR, "selection.json")

  REGISTERED_MODEL_NAME = "fraud-detector"
  RELEASE_ALIAS = "staging"


  def main():
      if not os.path.exists(SELECTION_JSON):
          sys.exit(
              f"[register] {SELECTION_JSON} missing — the select stage "
              "has not produced a selection yet."
          )
      with open(SELECTION_JSON) as f:
          selection = json.load(f)

      mlflow.set_tracking_uri(TRACKING_URI)
      client = MlflowClient()

      model_uri = f"runs:/{selection['run_id']}/model"
      version = mlflow.register_model(model_uri, REGISTERED_MODEL_NAME)

      client.set_registered_model_alias(
          REGISTERED_MODEL_NAME, RELEASE_ALIAS, version.version,
      )
      print(
          f"[register] {REGISTERED_MODEL_NAME} v{version.version} "
          f"aliased as {RELEASE_ALIAS!r}"
      )


  if __name__ == "__main__":
      main()
  ```

- Update `Makefile`

  Change pipeline order

  ```make
  .PHONY: train-pipeline clean

  # xFusionCorp Industries — Fraud Detection Training Pipeline.
  # Usage: make train-pipeline

  train-pipeline:
      python3 src/validate_data.py
      python3 src/tune.py
      python3 src/select_model.py
      python3 src/register.py
      python3 src/report.py

  clean:
      rm -rf models/ reports/
  ```

- Verify according to specified requirements
