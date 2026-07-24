### Task

The xFusionCorp Industries ML platform team has deployed a new fraud-detection model into production, utilizing an A/B router to manage traffic. The traffic distribution is set at 80% for the stable `MODEL_V1` and 20% for the candidate `MODEL_V2`. Each response from the server includes a `model_version` field, enabling downstream monitoring to accurately attribute each prediction to the corresponding model. The `ab_server.py` scaffold, located at `/root/code/serving/`, is responsible for loading both models and parsing incoming requests; however, the routing logic has yet to be implemented. Your task is to develop the A/B routing functionality in `ab_server.py`, ensuring that approximately 80% of traffic is directed to `MODEL_V1`, 20% to `MODEL_V2`, and that all responses correctly indicate which model provided the prediction.

1. Flask is installed at startup (not part of the lab image by default). Two model versions are pre-trained: `model_v1.pkl` (10-tree RandomForest) and `model_v2.pkl` (50-tree RandomForest). Both live under `/root/code/serving/`.

2. The project layout under `/root/code/serving/`:
   - `model_v1.pkl` + `model_v2.pkl` – The two model versions the router multiplexes between. Correct.
   - `ab_server.py` – Flask app. `/health`, both model loads, and the request-body parsing in `POST /predict` are wired; the routing logic (split, model selection, response) is left as a `TODO` to author.

3. The end state must include:
   - `ab_server.py` splits traffic 80 % to `MODEL_V1` and 20 % to `MODEL_V2`.
   - Every response to `POST /predict` carries both `is_fraud` and `model_version`; `model_version` is `"v1"` or `"v2"`.
   - Over a batch of 200 requests, roughly 160 land on v1 (±20) and roughly 40 land on v2 (±20).

Flask reads the JSON body via `request.get_json()`; the scaffold already handles this.

### Solution

- Change directory

  ```bash
  cd serving/
  ```

- Update the `ab_server.py`

  ```python
  """A/B-testing Flask server for the fraud-detection model.

  Loads two model versions (`model_v1.pkl` + `model_v2.pkl`) and
  routes incoming traffic between them: the bulk of requests must
  reach the stable v1, a small slice goes to the candidate v2 so
  its predictions can be compared against v1 for drift.

  The release checklist requires an 80 / 20 split (v1 / v2) and a
  `model_version` field on every response so downstream monitoring
  can attribute each prediction to the model that produced it.
  """
  import random

  import joblib
  import numpy as np
  from flask import Flask, jsonify, request

  MODEL_V1 = joblib.load("/root/code/serving/model_v1.pkl")
  MODEL_V2 = joblib.load("/root/code/serving/model_v2.pkl")

  app = Flask(__name__)


  @app.route("/health")
  def health():
      return jsonify({"status": "ok"}), 200


  @app.route("/predict", methods=["POST"])
  def predict():
      payload = request.get_json() or {}
      features = np.array([[
          float(payload.get("amount", 0.0)),
          int(payload.get("hour", 0)),
          int(payload.get("num_tx_past_day", 0)),
      ]])

      # TODO: implement the A/B routing:
      #   1. route ~80% of traffic to MODEL_V1 (label "v1") and ~20% to
      #      MODEL_V2 (label "v2") — e.g. branch on random.random() < 0.8
      #   2. score `features` with the chosen model: model.predict(...)[0]
      #   3. return {"is_fraud": <int>, "model_version": "v1" | "v2"} so
      #      downstream monitoring can attribute each prediction to the
      #      model that produced it
      if random.random() < 0.8:
          model = MODEL_V1
          model_version = "v1"
      else:
          model = MODEL_V2
          model_version = "v2"

      prediction = model.predict(features)[0]

      return jsonify({
          "is_fraud": int(prediction),
          "model_version": model_version
      }), 200



  if __name__ == "__main__":
      app.run(host="0.0.0.0", port=8085)
  ```

- Run the script

  ```bash
  python3 ab_server.py
  ```
