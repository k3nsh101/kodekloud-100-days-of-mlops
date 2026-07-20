### Task

The xFusionCorp Industries ML platform team has developed a fraud-detection model that is accessible via a FastAPI service. This service features an auto-generated Swagger UI available at `/docs`. In the draft `app.py` file located at `/root/code/serving/`, the pre-trained model is loaded and the `/health` endpoint is implemented. However, the `PredictRequest` schema and the `POST /predict` handler have not yet been completed.

Your task is to finalize the `app.py` file by defining the typed request model and implementing the `/predict` handler. Additionally, ensure the server operates on port `8085`, and verify that it successfully scores transactions through the Swagger UI while rejecting any invalid input.

1. FastAPI + uvicorn are baked into the lab image. The **FastAPI Swagger UI** button opens `/docs` once the server is running on port `8085`.

2. The project layout under `/root/code/serving/`:
   - `model.pkl` – Deterministic RandomForest trained at startup on the shared `amount / hour / num_tx_past_day → is_fraud` synthetic dataset.
   - `train.csv` – The 10-row source used to fit the model.
   - `app.py` – FastAPI app. `/health`, the root→`/docs` redirect, the response models, and `GET /last-predictions` are wired. Two things are left as `TODO`s to author:
     - **TODO 1** – the `PredictRequest` model's three typed fields (`amount`, `hour`, `num_tx_past_day`), with types and range constraints that drive FastAPI's request validation + Swagger schema.
     - **TODO 2** – the `POST /predict` handler body (which returns `501` until authored): build the feature row, score it with `MODEL.predict(...)`, record it, and return `PredictResponse`.

3. The end state must include:
   - The Swagger UI at `/docs` is reachable (the **FastAPI Swagger UI** button loads it).
   - `POST /predict` with a valid payload returns `{"is_fraud": 0}` or `{"is_fraud": 1}` (HTTP 200).
   - Two distinct payloads return different `is_fraud` values – The handler reads the posted features.
   - `POST /predict` with an out-of-range field (e.g. `hour: 25`) returns HTTP `422` – The typed model rejects invalid input before the handler runs.

Suggested payloads: `{"amount": 3200, "hour": 23, "num_tx_past_day": 5}` (high-value, late-night—expected to flag fraud); `{"amount": 25.5, "hour": 10, "num_tx_past_day": 1}` (low-value, daytime—expected to pass).

### Solution

- Change directory

  ```bash
  cd serving/
  ```

- Update the `app.py`

  ```python
  """FastAPI fraud-detection prediction service.

  Exposes:
    - GET  /health               — liveness check.
    - POST /predict              — score one transaction.
    - GET  /last-predictions     — audit log of every POST /predict
                                   received since the server booted.
    - GET  /docs                 — auto-generated Swagger UI.
  """
  from typing import List

  import joblib
  import numpy as np
  from fastapi import FastAPI, HTTPException
  from fastapi.responses import RedirectResponse
  from pydantic import BaseModel, Field

  MODEL_PATH = "/root/code/serving/model.pkl"
  MODEL = joblib.load(MODEL_PATH)

  app = FastAPI(
      title="xFusionCorp Fraud-Detection API",
      description=(
          "Synchronous inference for the fraud-detection RandomForest. "
          "Submit predictions interactively from this Swagger UI with "
          "the `Try it out` panel on `/predict`."
      ),
      version="1.0.0",
  )

  prediction_history: List[dict] = []


  class PredictRequest(BaseModel):
      # TODO 1: declare the three typed request fields. Their types (and
      #         constraints) drive FastAPI's automatic request validation
      #         and the Swagger schema — an out-of-range value is rejected
      #         with HTTP 422 before the handler runs:
      #           amount           : float
      #           hour             : int constrained to 0-23  (ge=0, le=23)
      #           num_tx_past_day  : int, non-negative         (ge=0)
      #         Use pydantic Field(...) for the constraints, e.g.
      #           hour: int = Field(..., ge=0, le=23)
      amount: float
      hour: int = Field(..., ge=0, le=23)
      num_tx_past_day: int = Field(..., ge=0)


  class PredictResponse(BaseModel):
      is_fraud: int = Field(..., description="1 if the model flags fraud, else 0.")


  class HistoryEntry(BaseModel):
      amount: float
      hour: int
      num_tx_past_day: int
      is_fraud: int


  class HistoryResponse(BaseModel):
      count: int
      predictions: List[HistoryEntry]


  @app.get("/", include_in_schema=False)
  def root():
      """Redirect the bare root to the Swagger UI so the platform's
      port-forward button lands directly on the interactive docs."""
      return RedirectResponse(url="/docs")


  @app.get("/health")
  def health():
      return {"status": "ok"}


  @app.post("/predict", response_model=PredictResponse)
  def predict(req: PredictRequest) -> PredictResponse:
      # TODO 2: author the prediction handler:
      #   1. build a feature row [[req.amount, req.hour, req.num_tx_past_day]]
      #      as a numpy array
      #   2. score it with MODEL.predict(...) and cast the result to int
      #   3. append the inputs + result to prediction_history (keys match
      #      HistoryEntry: amount, hour, num_tx_past_day, is_fraud)
      #   4. return PredictResponse(is_fraud=<int>)
      features = np.array([[req.amount, req.hour, req.num_tx_past_day]])

      is_fraud = int(MODEL.predict(features)[0])

      prediction_history.append({
          "amount": req.amount,
          "hour": req.hour,
          "num_tx_past_day": req.num_tx_past_day,
          "is_fraud": is_fraud,
      })

      return PredictResponse(is_fraud=is_fraud)


  @app.get("/last-predictions", response_model=HistoryResponse)
  def last_predictions() -> HistoryResponse:
      return HistoryResponse(
          count=len(prediction_history),
          predictions=[HistoryEntry(**p) for p in prediction_history],
      )
  ```

- Start the server

  ```bash
  uvicorn app:app --host 0.0.0.0 --port 8085
  ```

- Verify the result

  Open **FastAPI Swagger UI** using the button and verify it is reachable.

  Open a new terminal.
  The following should return `{"is_fraud":1}`

  ```bash
  curl -X POST http://localhost:8085/predict -H 'Content-Type: application/json' -d '{"amount":3200,"hour":23,"num_tx_past_day":5}'
  ```

  The following should return `{"is_fraud":0}`

  ```bash
  curl -X POST http://localhost:8085/predict -H 'Content-Type: application/json' -d '{"amount":25.5,"hour":10,"num_tx_past_day":1}'
  ```

  The following should return

  ```
  {"detail":[{"type":"less_than_equal","loc":["body","hour"],"msg":"Input should be less than or equal to 23","input":25,"ctx":{"le":23}}]}
  HTTP STATUS: 422
  ```

  ```bash
  curl -s -w "\nHTTP STATUS: %{http_code}\n" -X POST http://localhost:8085/predict -H 'Content-Type: application/json' -d '{"amount":2
  5.5,"hour":25,"num_tx_past_day":1}'
  ```
