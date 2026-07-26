### Task

The xFusionCorp Industries ML platform team operates an asynchronous fraud-detection scoring system, ensuring that the HTTP entry point responds within single-digit milliseconds while the model processes data in a background worker. The scaffold for this process, located at `/root/code/serving/async_app.py`, is designed to delegate tasks to a background worker and is intended to persist the results of each task in Redis. However, the implementation for storing results in Redis has not yet been completed.

Your objective is to implement the Redis round-trip within `async_app.py`. This involves storing each result in Redis after the worker has completed its task. In addition, you must ensure that the `GET /result/<task_id>` endpoint retrieves the stored results. The expected workflow is for clients to submit a request through `POST /predict-async`, then to subsequently poll the results using `GET /result/<task_id>`, which should return an `is_fraud` flag corresponding to the submitted payload.

1. Flask + redis-py are installed at startup. A Redis container named `async-redis` is already running on host port `6379`.

2. The project layout under `/root/code/serving/`:
   - `model.pkl` – Deterministic RandomForest trained at startup.
   - `async_app.py` – Flask app. The Redis connection, `/health`, `POST /predict-async` (returns a `task_id`, runs the model on a background thread), and the thread itself are wired. Two things are left as `TODOs` to author: the worker's result store in Redis, and the `GET /result/<task_id>` lookup that reads it back.

3. The end state must include:
   - `redis.Redis(host="localhost", port=6379, ...)` in `async_app.py`.
   - `GET /result/<task_id>` reads the stored value back from Redis and returns it as part of the JSON body.
   - `POST /predict-async` returns a JSON body carrying a `task_id`; after a short poll, `GET /result/<task_id>` returns a JSON body carrying an `is_fraud` flag of `0` or `1`.

The background worker stores results at keys shaped `result:<task_id>`, with a 600-second TTL.

### Solution

- Change directory

  ```bash
  cd serving/
  ```

- Update the `async_app.py`

  ```python
  """Flask async prediction server backed by Redis.

  Incoming `POST /predict-async` requests return a `task_id` and
  kick off prediction on a background thread. The worker writes the
  result to Redis under `result:<task_id>` with a 600-second TTL.
  Clients poll `GET /result/<task_id>` to retrieve the stored
  classification once the worker has finished.
  """
  import threading
  import time
  import uuid

  import joblib
  import numpy as np
  import redis
  from flask import Flask, jsonify, request

  MODEL = joblib.load("/root/code/serving/model.pkl")
  REDIS = redis.Redis(host="localhost", port=6379, decode_responses=True)

  RESULT_KEY = "result:{task_id}"
  RESULT_TTL_SECONDS = 600

  app = Flask(__name__)


  def _run_prediction(task_id: str, features) -> None:
      time.sleep(0.3)
      is_fraud = int(MODEL.predict(np.array([features]))[0])
      # TODO: persist the result so GET /result can retrieve it — set
      #       the Redis key RESULT_KEY.format(task_id=task_id) to
      #       is_fraud with a RESULT_TTL_SECONDS expiry
      #       (REDIS.set(key, value, ex=...)).
      key = RESULT_KEY.format(task_id=task_id)
      REDIS.set(key, is_fraud, ex=RESULT_TTL_SECONDS)


  @app.route("/health")
  def health():
      return jsonify({"status": "ok"}), 200


  @app.route("/predict-async", methods=["POST"])
  def predict_async():
      payload = request.get_json() or {}
      features = [
          float(payload.get("amount", 0.0)),
          int(payload.get("hour", 0)),
          int(payload.get("num_tx_past_day", 0)),
      ]
      task_id = uuid.uuid4().hex
      threading.Thread(
          target=_run_prediction, args=(task_id, features), daemon=True
      ).start()
      return jsonify({"task_id": task_id}), 202


  @app.route("/result/<task_id>")
  def result(task_id):
      # TODO: read the stored classification from Redis and return it —
      #       look up RESULT_KEY.format(task_id=task_id) with
      #       REDIS.get(...). If a value is present, return e.g.
      #       {"task_id": task_id, "is_fraud": int(value)}; if not yet
      #       ready, return a pending response (e.g. status "pending").
      key = RESULT_KEY.format(task_id=task_id)
      value = REDIS.get(key)

      if value is None:
          return jsonify({
              "task_id": task_id,
              "status": "pending"
          }), 200

      return jsonify({
          "task_id": task_id,
          "is_fraud": int(value)
      }), 200


  if __name__ == "__main__":
      app.run(host="0.0.0.0", port=8085)
  ```

- Run the script

  ```bash
  python3 async_app.py
  ```

- Start a new terminal and verify the results

  Verify the state

  ```bash
  curl http://localhost:8085/health
  ```

  The following returns a task_id.
  e.g. `{"task_id":"ca0ac8ec56be43f48e9489ab39bbb4d4"}`

  ```bash
  curl http://localhost:8085/predict-async -H 'Content-Type: application/json' -d '{"amount":3200,"hour":23,"num_tx_past_day":5}'
  ```

  After a while the following should return `is_fraud` 1 or 0.
  e.g. `{"is_fraud":1,"task_id":"ca0ac8ec56be43f48e9489ab39bbb4d4"}`

  ```bash
  curl http://localhost:8085/result/<task_id>
  ```
