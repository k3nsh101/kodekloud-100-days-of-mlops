### Task

The xFusionCorp Industries ML platform team runs a parallel-training bake-off on the fraud-detection model—the same estimator is trained twice, once on a single worker and once across every available CPU, and the MLflow Compare view surfaces the wall-time gap. A draft script exists at `/root/code/fraud-detection/src/models/train_parallel.py`, but running it currently produces near-identical wall times for the 'serial' and 'parallel' runs, and the Compare view cannot distinguish the two configurations. Your task is to correct the script so the second run genuinely runs in parallel and every MLflow run records the number of workers it actually used.

1. The MLflow tracking server is already running on port `5000`. The **MLflow UI** button at the top of the lab can be opened to confirm—the dashboard loads with an empty `parallel-training` experiment.

2. The project layout under `/root/code/fraud-detection/`:
   - `data/train.csv` – A 5000-row synthetic binary-classification dataset (imbalanced roughly 70 / 30). Larger than the 200-row dataset used earlier in the section because the `n_jobs` speedup is only visible once there is enough work per tree.
   - `src/models/train_parallel.py` – The bake-off script. Data loading, MLflow experiment setup, wall-time measurement, `metrics.training_time_seconds` logging, and model persistence to `models/model.pkl` are already wired; corrections are required.

3. Open `src/models/train_parallel.py` in the VS Code editor, correct every issue that keeps the bake-off from meeting the release checklist, save, and run the script once.

4. The end state must include:
   - At least two runs exist in the `parallel-training` experiment on MLflow. Across the two runs, `params.n_jobs` takes the values `1` and `-1` (no run still carries the hardcoded `"all"`).
   - Every run carries `metrics.training_time_seconds`, and the `n_jobs = -1` run is measurably faster than the `n_jobs = 1` run (at least 10 %).
   - A pickled model at `/root/code/fraud-detection/models/model.pkl`.

### Solution

- Update the `/root/code/fraud-detection/src/models/train_parallel.py`

  ```python
  """Parallel training bake-off for the fraud-detection model.

  Trains the same RandomForestClassifier twice — once on a single
  worker, once across every available CPU — so the two configurations
  can be compared side-by-side in the MLflow UI. Every run logs the
  measured wall time as `metrics.training_time_seconds` and the
  `n_jobs` value actually used under `params.n_jobs`.

  Every non-end-state concern is correctly wired — data loading, CV
  split, MLflow experiment setup, model persistence. Adjust the
  `N_JOBS_VALUES` list and the `mlflow.log_param` call so the second
  run actually runs in parallel and the logged `n_jobs` parameter
  distinguishes the two runs in the UI.
  """
  import os
  import time

  import joblib
  import mlflow
  import pandas as pd
  from sklearn.ensemble import RandomForestClassifier

  TRACKING_URI = "http://localhost:5000"
  EXPERIMENT = "parallel-training"
  TRAIN_CSV = "/root/code/fraud-detection/data/train.csv"
  MODEL_PATH = "/root/code/fraud-detection/models/model.pkl"

  N_ESTIMATORS = 200
  RANDOM_STATE = 42

  N_JOBS_VALUES = [1, -1]


  def main():
      mlflow.set_tracking_uri(TRACKING_URI)
      mlflow.set_experiment(EXPERIMENT)

      df = pd.read_csv(TRAIN_CSV)
      X = df.drop(columns=["is_fraud"])
      y = df["is_fraud"]

      last_model = None
      for n_jobs in N_JOBS_VALUES:
          run_name = "serial" if n_jobs == 1 else "parallel"
          with mlflow.start_run(run_name=run_name):
              mlflow.log_param("n_jobs", n_jobs)
              mlflow.log_param("n_estimators", N_ESTIMATORS)

              model = RandomForestClassifier(
                  n_estimators=N_ESTIMATORS,
                  random_state=RANDOM_STATE,
                  n_jobs=n_jobs,
              )
              start = time.perf_counter()
              model.fit(X, y)
              elapsed = time.perf_counter() - start

              mlflow.log_metric("training_time_seconds", elapsed)
              print(
                  f"[{run_name}] n_jobs={n_jobs}  "
                  f"training_time_seconds={elapsed:.3f}"
              )
              last_model = model

      os.makedirs(os.path.dirname(MODEL_PATH), exist_ok=True)
      joblib.dump(last_model, MODEL_PATH)
      print(f"Model saved to {MODEL_PATH}")


  if __name__ == "__main__":
      main()
  ```

- Run the script

  ```bash
  python3 /root/code/fraud-detection/src/models/train_parallel.py
  ```

- Verify from the **MLflow UI**
