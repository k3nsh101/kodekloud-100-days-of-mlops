### Task

The xFusionCorp Industries ML platform team evaluates fraud-detection candidates with k-fold cross-validation so every candidate is measured on multiple folds of an imbalanced dataset. A draft cross-validation scaffold exists at `/root/code/fraud-detection/src/models/cross_validate.py`, but its report does not match the release checklist and the fold strategy does not preserve the class ratio. Your task is to correct the scaffold so the cross-validation report lands in the expected shape.

1. The MLflow tracking server is already running on port `5000`. The **MLflow UI** button at the top of the lab can be opened to confirm—the dashboard loads with an empty `fraud-detection-cv` experiment.

2. The project layout under `/root/code/fraud-detection/`:
   - `data/train.csv` – A pre-generated 200-row synthetic binary-classification dataset with an imbalanced class split (roughly 70 / 30). Do not regenerate it.
   - `src/models/cross_validate.py` – The cross-validation scaffold. Every concern other than the splitter and the aggregate schema is correctly wired: fold iteration, per-fold metric computation, nested MLflow runs under a parent, JSON persistence, and artefact logging.
   - `reports/` – Where the cross-validation report must land.

3. Open `src/models/cross_validate.py` in the VS Code editor, correct the two problems that keep the report from meeting the release checklist, save, and run the script.

4. The end state must include:
   - A file at `/root/code/fraud-detection/reports/cv\*results.json` (absolute path, inside the project's reports directory).
   - That JSON contains exactly these seven top-level keys: `mean_accuracy`, `std_accuracy`, `mean_f1`, `std_f1`, `mean_roc_auc`, `std_roc_auc`, `folds`. Every `mean_*` and `std_*` value is numeric.
   - The `folds` value is a list of five per-fold dictionaries; each carries the keys `fold`, `accuracy`, `f1`, `roc_auc`.
   - The cross-validation splitter is stratification-aware – Each fold preserves the dataset's class ratio.
   - One parent MLflow run in the `fraud-detection-cv` experiment with five nested children (`fold-1` through `fold-5`), each logging the per-fold metrics.

`StratifiedKFold` is already imported at the top of the scaffold—no new imports are required. The fix is confined to the CV splitter and the aggregate dict.

### Solution

- Update `root/code/fraud-detection/src/models/cross_validate.py`

  Use `StratifiedKFold` instead of `KFold`. Add `std_*` to the aggregate dictionary.

  ```python
  """Cross-validation evaluator for the fraud-detection model.

  Runs k-fold cross-validation, logs every fold as a nested MLflow run
  under a single parent, and writes an aggregate report at
  `reports/cv_results.json`.

  Every non-end-state concern is correctly wired — fold iteration,
  metric computation, parent + child MLflow runs, artefact logging.
  Adjust the CV splitter and the aggregate dict so the report
  matches the schema the release checklist requires.
  """
  import os
  import json
  import numpy as np
  import pandas as pd
  import mlflow
  import mlflow.sklearn
  from sklearn.ensemble import RandomForestClassifier
  from sklearn.model_selection import KFold, StratifiedKFold
  from sklearn.metrics import accuracy_score, f1_score, roc_auc_score

  TRAIN_CSV = "/root/code/fraud-detection/data/train.csv"
  REPORTS_DIR = "/root/code/fraud-detection/reports"
  CV_RESULTS_JSON = os.path.join(REPORTS_DIR, "cv_results.json")
  N_SPLITS = 5
  EXPERIMENT_NAME = "fraud-detection-cv"


  def main():
      df = pd.read_csv(TRAIN_CSV)
      X = df.drop(columns=["is_fraud"])
      y = df["is_fraud"]

      cv = StratifiedKFold(n_splits=N_SPLITS, shuffle=True, random_state=42)

      mlflow.set_tracking_uri("http://localhost:5000")
      mlflow.set_experiment(EXPERIMENT_NAME)

      fold_results = []

      with mlflow.start_run(run_name="cv-parent"):
          mlflow.log_param("n_splits", N_SPLITS)
          mlflow.log_param("cv_type", type(cv).__name__)

          for fold_idx, (train_idx, test_idx) in enumerate(cv.split(X, y), start=1):
              X_train, X_test = X.iloc[train_idx], X.iloc[test_idx]
              y_train, y_test = y.iloc[train_idx], y.iloc[test_idx]

              model = RandomForestClassifier(
                  n_estimators=100, max_depth=5, random_state=42
              )
              model.fit(X_train, y_train)
              preds = model.predict(X_test)
              proba = model.predict_proba(X_test)[:, 1]

              fold = {
                  "fold": fold_idx,
                  "accuracy": round(accuracy_score(y_test, preds), 6),
                  "f1": round(f1_score(y_test, preds), 6),
                  "roc_auc": round(roc_auc_score(y_test, proba), 6),
              }
              fold_results.append(fold)

              with mlflow.start_run(run_name=f"fold-{fold_idx}", nested=True):
                  mlflow.log_param("fold", fold_idx)
                  mlflow.log_metric("accuracy", fold["accuracy"])
                  mlflow.log_metric("f1", fold["f1"])
                  mlflow.log_metric("roc_auc", fold["roc_auc"])

          acc_vals = [r["accuracy"] for r in fold_results]
          f1_vals = [r["f1"] for r in fold_results]
          auc_vals = [r["roc_auc"] for r in fold_results]

          aggregate = {
              "mean_accuracy": round(float(np.mean(acc_vals)), 6),
              "mean_f1": round(float(np.mean(f1_vals)), 6),
              "mean_roc_auc": round(float(np.mean(auc_vals)), 6),
              "std_accuracy": round(float(np.std(acc_vals)), 6),
              "std_f1": round(float(np.std(f1_vals)), 6),
              "std_roc_auc": round(float(np.std(auc_vals)), 6),
              "folds": fold_results,
          }

          mlflow.log_metric("mean_accuracy", aggregate["mean_accuracy"])
          mlflow.log_metric("mean_f1", aggregate["mean_f1"])
          mlflow.log_metric("mean_roc_auc", aggregate["mean_roc_auc"])

          os.makedirs(REPORTS_DIR, exist_ok=True)
          with open(CV_RESULTS_JSON, "w") as f:
              json.dump(aggregate, f, indent=2, sort_keys=False)
          mlflow.log_artifact(CV_RESULTS_JSON)

      print(f"aggregate: {aggregate}")
      print(f"report: {CV_RESULTS_JSON}")


  if __name__ == "__main__":
      main()
  ```

- Run the script

  ```bash
  python3 /root/code/fraud-detection/src/models/cross_validate.py
  ```

- Verify the `/root/code/fraud-detection/reports/cv_results.json` is created and follows the required format

- Verify via the **MLflow UI**
