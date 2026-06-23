### Task

The xFusionCorp Industries ML platform team runs fraud-detection training as a four-stage pipeline—preprocess, featurize, train, evaluate—orchestrated by a single script that logs the end-to-end run to MLflow. A pre-staged pipeline is already in place, but the stage-chain invariant is broken: the pipeline currently produces a feature matrix that does not reflect the upstream drop-and-clean work. Your task is to correct the stage wiring so every stage reads from its immediate predecessor and one MLflow run captures the full pipeline.

1. The MLflow tracking server is already running on port `5000`. The **MLflow UI** button at the top of the lab can be opened to confirm—the dashboard loads with an empty `training-pipeline` experiment.

2. The project layout under `/root/code/fraud-detection/`:
   - `data/raw/train.csv` – The same 200-row synthetic binary-classification dataset the rest of the Training section uses (imbalanced roughly 70 / 30).
   - `configs/pipeline_config.yaml` – Declares the data paths, model hyperparameters, output paths, and MLflow settings every stage consumes. Correct and must remain intact.
   - `src/preprocess.py`, `src/featurize.py`, `src/train.py`, `src/evaluate.py` – The four pipeline stages. `preprocess.py` drops negligible-amount rows (`amount < 50`) and duplicates before writing the processed CSV. The four stages are wired through the config's `data:` paths.
   - `run_pipeline.py` – The orchestrator that executes the four stages in order and logs one MLflow run with the config-driven parameters and the final evaluation metrics. Correct and requires no edits.

3. Identify the stage whose input path breaks the chain, correct the wiring in the VS Code editor, save, and run `python3 run_pipeline.py` once from the project root.

4. The end state must include:
   - The row count of `data/features/features.csv` equals the row count of `data/processed/train_clean.csv` and is strictly less than the 200-row raw CSV.
   - `models/model.pkl` and `reports/evaluation.json` are written and the report carries `accuracy`, `f1`, and `roc_auc` as numeric values.
   - Exactly one MLflow run exists in the `training-pipeline` experiment, carrying `params.model_type`, `params.n_estimators`, `params.max_depth`, and the three evaluation metrics.

### Solution

- Change the `featurize.py`.

  Change the `input_path` to `processed_path`

  ```python
  """Stage 2 — Featurize.

  Reads the upstream stage's output, engineers one derived column
  (`amount_log = log1p(amount)`), and writes the feature matrix to
  `data/features/features.csv` for the training stage to consume.

  Every concern other than the input wiring is correctly in place —
  feature engineering, column preservation, on-disk layout. Adjust
  the input source so the stage-chain invariant holds: the row count
  out of this stage must match the row count the preprocess stage
  produced.
  """
  import os

  import numpy as np
  import pandas as pd
  import yaml

  os.chdir("/root/code/fraud-detection")

  with open("configs/pipeline_config.yaml") as f:
      config = yaml.safe_load(f)

  input_path = config["data"]["processed_path"]
  features_path = config["data"]["features_path"]

  df = pd.read_csv(input_path)
  df["amount_log"] = np.log1p(df["amount"])

  os.makedirs(os.path.dirname(features_path), exist_ok=True)
  df.to_csv(features_path, index=False)

  print(
      f"[featurize] input={input_path}  rows={len(df)}  "
      f"columns={len(df.columns)}"
  )
  ```

- Run pipeline

  ```bash
  cd fraud-detection/
  python3 run_pipeline.py
  ```

- Verify via the **MLflow UI** according to the requirements specified in the 4th step.
