### Task

After training a model, the xFusionCorp Industries ML team wants DVC to surface metrics through `dvc metrics show` and the DVC extension's METRICS view. The fraud-detection pipeline already trains a model and writes a `metrics.json`, but DVC does not recognise the file as a metric. Wire it in correctly.

1. A project exists at `/root/code/fraud-detection/` with a three-stage DVC pipeline (`process_data`, `split_data`, `train`). The train stage runs `src/models/train.py`, which writes the model to `models/model.pkl` and metrics to `metrics.json`. Do not modify the Python files.

2. The `train` stage in `dvc.yaml` must declare `metrics.json` as a DVC metric output, not as a regular file output. The metric must be declared with `cache: false` so the JSON lives in Git for diff history rather than in the DVC cache.

3. Re-run the pipeline with `dvc repro` so the metric registration takes effect.

4. After your changes, `dvc metrics show` must report the `accuracy` and `f1_score` values from `metrics.json`.

The DVC extension's **METRICS** section under the DVC view will surface the same values directly in the editor once the metric is registered.

### Solution

- Change directory

  ```bash
  cd fraud-detection
  ```

- Update the `dvc.yaml`

  ```yaml
  stages:
    process_data:
      cmd: python src/data/process_data.py
      deps:
        - data/raw/transactions.csv
        - src/data/process_data.py
      outs:
        - data/processed/clean_transactions.csv

    split_data:
      cmd: python src/data/split_data.py
      deps:
        - data/processed/clean_transactions.csv
        - src/data/split_data.py
      outs:
        - data/processed/train.csv
        - data/processed/test.csv

    train:
      cmd: python src/models/train.py
      deps:
        - data/processed/train.csv
        - src/models/train.py
      outs:
        - models/model.pkl

    metrics:
      - metrics.json:
          cache: false
  ```

- Run `dvc repro` and verify that `metrics.json` is created.

- Verify by running `dvc metrics show`. It should display the metrics with columns `Path`, `accuracy`, and `f1_score`
