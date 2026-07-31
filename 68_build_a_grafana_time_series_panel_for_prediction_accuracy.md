### Task

The xFusionCorp Industries ML platform team monitors the rolling prediction accuracy of the fraud-detection model using a Grafana dashboard. The monitoring stack is operational, with the Flask metric-emitter exposing `prediction_accuracy` as a gauge. Prometheus is configured to scrape this metric, and the Prometheus datasource is already pre-provisioned in Grafana. Your objective is to create a Grafana dashboard featuring a time-series panel that visualizes `prediction_accuracy` over time.

1. The Grafana UI is running on port `3000`. The Grafana button opens the login page. Admin credentials: `admin` / `grafana2026`. The **Prometheus** datasource is already wired (pre-provisioned at startup—no datasource form to fill).

2. Available `metrics` on the Prometheus datasource include:
   - `prediction_accuracy` – The gauge for this task.
   - `flask_http_request_total{version, endpoint, method}` – Counter.
   - `data_drift_score{column}` – Per-column drift gauge.
   - `model_inference_duration_seconds` – Latency histogram.

3. The end state must include:
   - Grafana's **Data sources** list shows a provisioned Prometheus datasource (pre-staged).
   - `GET /api/search?type=dash-db` returns at least one user-created dashboard.
   - That dashboard carries at least one panel whose `type` is `timeseries`.
   - At least one panel across your dashboards has a Prometheus query that references `prediction_accuracy`.

### Solution

- Log in to the Grafana dashboard

- Create a dashboard

  ```
  Dashboards -> Create dashboard
  ```

- Configure panel

  ```
  Add panel -> Configure visualization -> Select metric prediction_accuracy
  ```

  <img src="./assets/assets_68/result.png" alt="created dashboard" />

  <br />
