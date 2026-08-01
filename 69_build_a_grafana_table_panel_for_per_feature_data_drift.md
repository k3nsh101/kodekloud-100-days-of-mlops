### Task

The xFusionCorp Industries ML platform team monitors per-feature data drift for the fraud-detection model using a Grafana table. This table presents one row for each feature and one column for each drift score, allowing reviewers to easily scan the entire feature set. The monitoring stack is operational; the Flask metric-emitter exposes `data_drift_score` as a labeled gauge (with one time-series per feature), Prometheus is actively scraping the data, and Grafana has the Prometheus datasource pre-provisioned. Your objective is to create a Grafana dashboard featuring a **Table** panel that displays the `data_drift_score` values for each column.

1. The Grafana UI is running on port `3000`. The **Grafana button** opens the login page. Admin credentials: `admin` / `grafana2026`. The **Prometheus** datasource is pre-provisioned.

2. Metrics available on the Prometheus datasource:
   - `data_drift_score{column="amount"}`, `data_drift_score{column="hour"}`, `data_drift_score{column="num_tx_past_day"}` – The gauge for this task, one series per feature column.
   - `prediction_accuracy`, `flask_http_request_total{version, endpoint, method}`, `model_inference_duration_seconds` – The other signals from the shared metric-emitter.

3. The end state must include:
   - `GET /api/search?type=dash-db` returns at least one user-created dashboard.
   - At least one panel across your dashboards has `type: table`.
   - At least one of those panel's Prometheus targets references `data_drift_score`.
   - Querying `data_drift_score` through Grafana's datasource proxy returns non-empty Prometheus series whose labels include `column` – Confirming the table has per-feature rows to render.

A per-feature drift view answers 'which input shifted?' at a glance — a Table panel is the natural shape for it, one row per feature carrying that feature's latest drift score.

### Solution

- Log in to Grafana

- Create a dashboard

  ```
  Dashboards -> Create dashboard
  ```

- Configure panel

  ```
  Add panel -> Configure visualization
  Select metric as `data_drift_score`.
  Select options -> Type as `Instant` to get the lastest drift score.
  Set the panel type as `Table`
  ```

  <img src="./assets/assets_69/query_settings.png" alt="query settings" />

  <br />

  Add Transformation `Series to rows`

  <img src="./assets/assets_69/transformation_settings.png" alt="transformation settings" />

- Save the dashboard
