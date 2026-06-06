### Task

A xFusionCorp Industries data scientist has trained three candidate models on the same problem and logged them to the `model-comparison` experiment. Your task is to review the candidates side by side in the **MLflow UI** and explicitly mark the winning run so downstream tooling can pick it up.

1. The MLflow tracking server is already running on port `5000` and the `model-comparison` experiment has been pre-populated with three runs, each named after its algorithm (`RandomForest`, `GradientBoosting`, `LogisticRegression`) and carrying `accuracy` and `f1_score` metrics. The runs can be viewed via the **MLflow UI** button → `model-comparison` experiment.

2. Using the **MLflow UI**, inspect the three runs side by side and identify the winner by `metrics.f1_score`.
   - The run with the highest `f1_score` must carry a run-level tag: key `production-candidate`, value `true`.
   - Neither of the other two runs may carry a `production-candidate` tag.

The result can be confirmed in the **MLflow UI**: the `model-comparison` experiment lists three runs, and only the top-`f1_score` run shows the `production-candidate` tag on its detail page.

### Solution

- Go to the `model-comparison` experiment runs

  ```
  Experiments -> model-comparison -> Training runs
  ```

  <img src="./assets/assets_26/runs.png" alt="runs" />

  <br />

- Add `f1_score column` by adding a tick for the `f1_score` in the `Columns` menu

  <img src="./assets/assets_26/add_metric_column.png" alt="add metric column" />

  <br />

  <img src="./assets/assets_26/view_with_f1_score.png" alt="view with f1 score" />

  <br />

- Compare the `f1_score` values and add the `production-candidate` key to the run with highest `f1_score`

  ```
  Click on GradientBoosting -> Tags -> Add Tags
  ```

  <img src="./assets/assets_26/add_tag.png" alt="add tag" />

  <br />
