### Task

The xFusionCorp Industries ML platform team tunes fraud-detection hyperparameters with Optuna and inspects the full search in the MLflow Compare view. A draft tuner exists at `/root/code/fraud-detection/src/models/tune.py`, but the search optimises in the wrong direction and no trials ever land on the tracking server. Your task is to correct the tuner so each of the 20 trials is visible in MLflow and the saved best configuration is actually the highest-F1 candidate.

1. The MLflow tracking server is already running on port `5000`. The **MLflow UI** button at the top of the lab can be opened to confirm—the dashboard loads with an empty hyperopt-tuning experiment.

2. The project layout under `/root/code/fraud-detection/`:
   - `data/train.csv` – The same 200-row synthetic binary-classification dataset Day 34 uses (imbalanced roughly 70 / 30).
   - `src/models/tune.py` – The Optuna tuner scaffold. Fold iteration, metric averaging, Optuna study creation, and YAML persistence are already wired; corrections are required.
   - `configs/` – Where `best_params.yaml` is written after the search completes.

3. Open `src/models/tune.py` in the VS Code editor, correct every issue that keeps the search from meeting the release checklist, save, and run the script once.

4. The end state must include:
   - At least 20 runs exist in the `hyperopt-tuning` experiment on MLflow. Every run carries `params.n_estimators`, `params.max_depth`, and `metrics.f1_score`.
   - A YAML file at `/root/code/fraud-detection/configs/best_params.yaml` with exactly two keys: `n_estimators` (integer in the range `[50, 500]`) and `max_depth` (integer in the range `[3, 20]`).
   - The Optuna study is directed such that `best_params` corresponds to the highest-F1 trial in the search space – Not the lowest.

### Solution

- Update the `root/code/fraud-detection/src/models/tune.py`
  Add logging in the objective function and change direction minimize to maximize

  ```python
  """Hyperparameter tuner for the fraud-detection RandomForest.

  Uses Optuna to sample `n_estimators` and `max_depth` across 20
  trials, evaluates each candidate via 3-fold stratified cross-
  validation on the synthetic training set, and writes the best
  configuration to `configs/best_params.yaml`.

  Every trial must land in the MLflow `hyperopt-tuning` experiment
  as an independent run so the full search can be inspected in the
  Compare view — the per-trial hyperparameters logged as run
  parameters and the resulting mean f1 as a run metric named
  `f1_score`.

  The scaffold below already implements fold iteration, metric
  averaging, and the Optuna / YAML wiring. Two corrections remain
  before the search produces the configuration the release checklist
  requires.
  """
  import os
  import yaml
  import numpy as np
  import pandas as pd
  import optuna
  import mlflow
  from sklearn.ensemble import RandomForestClassifier
  from sklearn.model_selection import StratifiedKFold, cross_val_score

  TRAIN_CSV = "/root/code/fraud-detection/data/train.csv"
  CONFIGS_DIR = "/root/code/fraud-detection/configs"
  BEST_PARAMS_YAML = os.path.join(CONFIGS_DIR, "best_params.yaml")
  EXPERIMENT_NAME = "hyperopt-tuning"
  N_TRIALS = 20
  N_SPLITS = 3

  mlflow.set_tracking_uri("http://localhost:5000")
  mlflow.set_experiment(EXPERIMENT_NAME)


  def objective(trial, X, y):
      n_estimators = trial.suggest_int("n_estimators", 50, 500)
      max_depth = trial.suggest_int("max_depth", 3, 20)

      model = RandomForestClassifier(
          n_estimators=n_estimators,
          max_depth=max_depth,
          random_state=42,
      )
      cv = StratifiedKFold(n_splits=N_SPLITS, shuffle=True, random_state=42)
      scores = cross_val_score(model, X, y, cv=cv, scoring="f1")
      score = float(np.mean(scores))

      with mlflow.start_run():
          mlflow.log_param("n_estimators", n_estimators)
          mlflow.log_param("max_depth", max_depth)
          mlflow.log_metric("f1_score", score)

      return score


  def main():
      df = pd.read_csv(TRAIN_CSV)
      X = df.drop(columns=["is_fraud"])
      y = df["is_fraud"]

      study = optuna.create_study(
          direction="maximize", study_name=EXPERIMENT_NAME
      )
      study.optimize(lambda trial: objective(trial, X, y), n_trials=N_TRIALS)

      os.makedirs(CONFIGS_DIR, exist_ok=True)
      with open(BEST_PARAMS_YAML, "w") as f:
          yaml.safe_dump(study.best_params, f, sort_keys=True)

      print(f"best params: {study.best_params}")
      print(f"best f1: {study.best_value:.6f}")
      print(f"wrote {BEST_PARAMS_YAML}")


  if __name__ == "__main__":
      main()
  ```

- Run the script

  ```bash
  python3 /root/code/fraud-detection/src/models/tune.py
  ```

- Verify that the `root/code/fraud-detection/configs/best_params.yaml` exists and **MLflow UI** shows the requirements

  <img src="./assets/assets_35/result.png" alt="results" />
