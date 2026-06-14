### Task

The xFusionCorp Industries ML platform team's release checklist requires a five-metric evaluation report for every candidate model, plus a confusion-matrix image, published to the project's `reports/` directory. A draft `evaluate.py` exists for the pre-trained fraud-detection model, but the report it produces does not satisfy the checklist. Your task is to correct the evaluator so the expected report lands in the right place.

1. The MLflow tracking server is already running on port `5000`. The **MLflow UI** button at the top of the lab can be opened to confirm—the dashboard loads with an empty `fraud-detection-eval` experiment.

2. The project layout under `/root/code/fraud-detection/`:
   - `data/test.csv` – 40-row held-out test set from an 80/20 stratified split.
   - `models/model.pkl` – A deterministic `RandomForestClassifier` pre-trained at lab startup. Do not retrain it.
   - `src/models/evaluate.py` – The evaluator draft (has bugs). Every concern other than the metrics report is correctly wired: the confusion-matrix rendering, the MLflow run, the artefact logging.
   - `reports/` – Where the metrics JSON and confusion-matrix image must land.

3. Open `src/models/evaluate.py` in the VS Code editor, correct everything that prevents the end state below from being reached, save, and run the script.

4. The end state must include:
   - A file at `/root/code/fraud-detection/reports/metrics.json` (absolute path, inside the project's reports directory).
   - That JSON contains exactly these five keys, each a numeric value: `accuracy`, `precision`, `recall`, `f1_score`, `auc_roc`.
   - A file at `/root/code/fraud-detection/reports/confusion_matrix.png`.
   - One MLflow run in the `fraud-detection-eval` experiment with the five metrics logged and both files attached as run artefacts.

The model file and test set are correct and must not be modified—the fix is confined to `evaluate.py`.

### Solution

- Update the `root/code/fraud-detection/scr/models/evaluate.py`

  Update the `METRICS_JSON` and `metrics` dictionary

  ```python
  """Evaluation script for the pre-trained fraud-detection model.

  Loads the serialised model + held-out test set, produces a metrics
  report and a confusion-matrix image, logs the results to MLflow.

  Three deliberate problems prevent the report from satisfying the
  lab's end state. Edit only the module-level constants and the
  metrics computation inside `main()`; every other part of the
  script — the confusion-matrix rendering, the MLflow wiring, the
  persistence calls — is correct.
  """
  import os
  import json
  import joblib
  import pandas as pd
  import matplotlib
  matplotlib.use("Agg")
  import matplotlib.pyplot as plt
  import mlflow
  from sklearn.metrics import (
      accuracy_score,
      precision_score,
      recall_score,
      f1_score,
      roc_auc_score,
      confusion_matrix,
  )

  MODEL_PATH = "/root/code/fraud-detection/models/model.pkl"
  TEST_CSV = "/root/code/fraud-detection/data/test.csv"
  REPORTS_DIR = "/root/code/fraud-detection/reports"

  METRICS_JSON = REPORTS_DIR + "/metrics.json"
  CONFUSION_PNG = os.path.join(REPORTS_DIR, "confusion_matrix.png")


  def render_confusion_matrix(cm, path):
      fig, ax = plt.subplots(figsize=(4, 4))
      ax.imshow(cm, cmap="Blues")
      for i in range(cm.shape[0]):
          for j in range(cm.shape[1]):
              ax.text(j, i, str(cm[i, j]), ha="center", va="center", color="black")
      ax.set_xlabel("predicted")
      ax.set_ylabel("actual")
      ax.set_title("confusion matrix")
      fig.tight_layout()
      fig.savefig(path, dpi=100)
      plt.close(fig)


  def main():
      model = joblib.load(MODEL_PATH)
      df = pd.read_csv(TEST_CSV)
      X = df.drop(columns=["is_fraud"])
      y = df["is_fraud"]

      preds = model.predict(X)
      proba = model.predict_proba(X)[:, 1]

      metrics = {
          "accuracy": round(accuracy_score(y, preds), 6),
          "f1_score": round(f1_score(y, preds), 6),
          "precision": round(precision_score(y, preds), 6),
          "recall": round(recall_score(y, preds), 6),
          "auc_roc": round(roc_auc_score(y, proba), 6)
      }

      os.makedirs(REPORTS_DIR, exist_ok=True)
      os.makedirs(os.path.dirname(METRICS_JSON) or ".", exist_ok=True)
      with open(METRICS_JSON, "w") as f:
          json.dump(metrics, f, indent=2, sort_keys=True)

      cm = confusion_matrix(y, preds)
      render_confusion_matrix(cm, CONFUSION_PNG)

      mlflow.set_tracking_uri("http://localhost:5000")
      mlflow.set_experiment("fraud-detection-eval")
      with mlflow.start_run(run_name="evaluation"):
          for key, value in metrics.items():
              mlflow.log_metric(key, value)
          mlflow.log_artifact(CONFUSION_PNG)
          if os.path.exists(METRICS_JSON):
              mlflow.log_artifact(METRICS_JSON)

      print(f"metrics: {metrics}")
      print(f"metrics JSON: {METRICS_JSON}")
      print(f"confusion matrix: {CONFUSION_PNG}")


  if __name__ == "__main__":
      main()
  ```

- Run the script

  ```bash
  python3 root/code/fraud-detection/scr/models/evaluate.py`
  ```

- Verfiy the results through **MLflow UI**

  ```
  MLflow UI -> fraud-detection-eval -> Evaluation runs -> Go to run
  ```

  Then view artifacts and model metrics

  <img src="./assets/assets_33/metrics.png" alt="metrics" />

  <br />

  <img src="./assets/assets_33/confusion_matrix.png" alt="confusion matrix" />

  <br />
