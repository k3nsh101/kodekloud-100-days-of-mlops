### Task

A xFusionCorp Industries data scientist has accumulated ten runs in the `fraud-detection` MLflow experiment. Your task is to triage those runs via the **MLflow UI**: mark the single best-performing candidate as the shortlisted model, and flag every clearly under-performing run for removal.

1. The MLflow tracking server is already running on port `5000`, and the `fraud-detection` experiment has been pre-populated with ten runs. The runs can be viewed via the **MLflow UI** button → `fraud-detection` experiment.

2. Using the MLflow UI, complete the triage below. The end state is what is tested—the path taken through the UI is not.
   - **Shortlist the best candidate**. Among all runs where `metrics.f1_score > 0.85`, the single run with the highest `f1_score` must carry a run-level tag: key `review-status`, value `shortlisted`.
   - **Reject the under-performers**. Every run where `metrics.f1_score < 0.75` must carry a run-level tag: key `review-status`, value `rejected`.

3. The other runs (those in the 0.75 ≤ f1 ≤ 0.85 band, and the second-best shortlisting candidate) must carry no `review-status` tag at all.

### Solution

- Click on the MLflow UI, and go to the experiment runs of the `fraud-detection` experiment.

  ```
  Experiments -> fraud-detection -> Evaluation runs
  ```

- Click on the run with the highest `f1_score` and go to `Overview` tab

  <img src="./assets/assets_23/highest_f1_score.png" alt="highest f1 score run" />

  <br />

  <img src="./assets/assets_23/run_ui.png" alt="run ui" />

  <br />

- Click on `Add tags` and add the `shortlisted` tag

  <img src="./assets/assets_23/add_shortlisted_tag.png" alt="add tag ui" />

  <br />

  <img src="./assets/assets_23/review_status_shortlisted.png" alt="review_status_shortlisted" />

  <br />

- Do the same for `under-performers`

  <img src="./assets/assets_23/review_status_rejected_1.png" alt="review_status_rejected_1" />

  <br />

  <img src="./assets/assets_23/review_status_rejected_2.png" alt="review_status_rejected_2" />

  <br />

- Verify other runs does not have the `review-status` tag
