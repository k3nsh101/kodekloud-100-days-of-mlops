### Task

The xFusionCorp Industries ML platform team has deployed the fraud-detection model using BentoML. The model is registered in BentoML's local store and is served over HTTP with the command `bentoml serve`, which automatically generates a Swagger UI at the server's root. Within the scaffold located at `/root/code/serving/service.py`, the modern `@bentoml.service` class API is utilized. This script loads the pre-registered `fraud_detector:latest` model from the store and defines the APIs, although the `POST /predict` handler remains unimplemented. Your objective is to implement the `/predict` handler to score a transaction using the loaded model. Additionally, you need to start the server on port `3000` and verify that it returns predictions.

1. The `fraud_detector` model is registered in BentoML's local store. The BentoML server is NOT pre-started — the BentoML UI button opens the Swagger surface once the server is running on port `3000`.

2. The project layout under `/root/code/serving/`:
   - `service.py` – BentoML service (`@bentoml.service` class `FraudService`). The model-store load (`bentoml.models.BentoModel` + `bentoml.sklearn.load_model` in `**init**`) and the `last_predictions` API are wired. The `predict` handler body is left as a `TODO` — it returns an error until authored. The handler takes the `amount`/`hour`/`num_tx_past_day` parameters and returns `{"is_fraud": <int>}`.
   - `train.csv` – The 10-row source used at startup to train and register the model with `bentoml.sklearn.save_model("fraud_detector", model)`.

3. The end state must include:
   - `bentoml models list` lists `fraud_detector` in the store.
   - `curl http://localhost:3000/` returns HTTP `200` – The Swagger UI is reachable once the server is running.
   - `POST /predict` with a valid payload returns `{"is_fraud": 0}` or `{"is_fraud": 1}`.
   - Two distinct payloads return different `is_fraud` values – The handler scores the posted features.

Suggested payloads: `{"amount": 3200, "hour": 23, "num_tx_past_day": 5}` (high-value, late-night—expected to flag fraud); `{"amount": 25.5, "hour": 10, "num_tx_past_day": 1}` (low-value, daytime).

### Solution

- Change directory

  ```bash
  cd serving/
  ```

- Update `service.py`

  ```python
  """BentoML service exposing the fraud-detection RandomForest.

  Loads `fraud_detector:latest` from the BentoML model store (saved at
  startup) and serves it with the modern `@bentoml.service` class API:
    - POST /predict           — score one transaction.
    - POST /last_predictions  — audit log of every POST /predict
                                handled since the server booted.

  `bentoml serve service:FraudService` starts the HTTP server on port
  3000 and auto-generates a Swagger UI at the server's root — the
  primary GUI surface for this lab.
  """
  from typing import Any, Dict, List

  import bentoml
  import numpy as np


  @bentoml.service(name="fraud_service")
  class FraudService:
      # Declare the model dependency from the store; BentoML resolves it
      # and makes it available for loading.
      bento_model = bentoml.models.BentoModel("fraud_detector:latest")

      def __init__(self) -> None:
          self.model = bentoml.sklearn.load_model(self.bento_model)
          self._history: List[Dict[str, Any]] = []

      @bentoml.api
      def predict(
          self, amount: float, hour: int, num_tx_past_day: int
      ) -> Dict[str, Any]:
          # TODO: author the prediction handler:
          #   1. build a feature row [[amount, hour, num_tx_past_day]] (numpy)
          #   2. score it with self.model.predict(...) and cast to int
          #   3. append the inputs + label to self._history (keys: amount,
          #      hour, num_tx_past_day, is_fraud)
          #   4. return {"is_fraud": <int>}
          features = np.array([[
              amount,
              hour,
              num_tx_past_day
          ]])

          prediction = int(self.model.predict(features)[0])

          self._history.append({
              "amount": amount,
              "hour": hour,
              "num_tx_past_day": num_tx_past_day,
              "is_fraud": prediction
          })

          return {"is_fraud": prediction}



      @bentoml.api
      def last_predictions(self) -> Dict[str, Any]:
          return {"count": len(self._history), "predictions": self._history}
  ```

- Verify the model exists

  ```bash
  bentoml models list
  ```

- Start the server

  ```bash
  bentoml serve service:FraudService --port 3000
  ```

- Open a new terminal and verify the result

  The following returns HTTP `200`

  ```bash
  curl -I http://localhost:3000
  ```

  The following returns `{"is_fraud": 1}`.

  ```bash
  curl -X POST http://localhost:3000/predict -H 'Content-Type: application/json' -d '{"amount":200,"hour":23,"num_tx_past_day":5}'
  ```

  The following returns `{"is_fraud": 0}`.

  ```bash
  curl -X POST http://localhost:3000/predict -H 'Content-Type: application/json' -d '{"amount":25.5,"hour":10,"num_tx_past_day":1}'
  ```
