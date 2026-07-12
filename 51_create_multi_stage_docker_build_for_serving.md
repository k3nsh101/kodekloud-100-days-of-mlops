### Task

The xFusionCorp Industries ML platform team has deployed a fraud-detection model as a Docker image. However, the current runtime image includes all packages required for the training phase and the training source itself, resulting in an unnecessarily large image. Your objective is to refactor the single-stage `Dockerfile` located at `/root/code/ml-serve/` into a multi-stage build. This should comprise a builder stage that trains the model and generates `model.pkl`, followed by a runtime stage that installs only the dependencies necessary for serving and copies the trained model from the builder stage.

1. The Docker daemon is already running. `docker version` can be run in a VS Code terminal to confirm.

2. The project layout under `/root/code/ml-serve/`:
   - `train_model.py` – Fits a 10-tree RandomForest on the shared 10-row synthetic fraud set and writes `/app/model.pkl` via `joblib.dump(...)`. Correct and must remain intact.
   - `serve.py` – Flask app loading the model and exposing `POST /predict` + `GET /health` on port `8080`. Correct and must remain intact.
   - `Dockerfile` – A single-stage build that installs `scikit-learn`, `pandas`, `numpy`, `joblib`, and `flask`, runs the trainer at build time to bake the model in, and serves. The reader rewrites this file.

3. The end state must include:
   - The Dockerfile carries at least two `FROM` instructions; the first is given a name (e.g. `AS builder`) so a later stage can reference it.
   - The builder stage produces `/app/model.pkl` (the trained model).
   - The runtime stage contains `/app/model.pkl` (copied out of the builder stage) and `serve.py`.
   - The runtime stage's `pip install` line installs only the four packages `serve.py` needs: `flask`, `joblib`, `numpy`, `scikit-learn`.
   - `docker images ml-serve:v1` lists the built image; `docker run --rm -p 8090:8080 ml-serve:v1` exposes `/health` returning `{"status": "ok"}` on port `8090j.

Multi-stage builds let you ship runtime images that carry only what the serving app needs — training dependencies and source files stay in the builder stage and are discarded. `docker build -t ml-serve:v1` . can be re-run as each change lands; Docker re-uses cached layers when only runtime-stage lines change.

### Solution

- Change directory

  ```bash
  cd ml-serve
  ```

- Update the `Dockerfile`

  ```docker
  FROM python:3.11-slim AS builder

  WORKDIR /app

  RUN pip install --no-cache-dir scikit-learn pandas numpy joblib

  COPY train_model.py .

  RUN python3 train_model.py


  FROM python:3.11-slim

  WORKDIR /app

  RUN pip install --no-cache-dir scikit-learn numpy joblib flask

  COPY --from=builder /app/model.pkl .
  COPY serve.py .

  EXPOSE 8080

  CMD ["python3", "serve.py"]
  ```

- Build the image

  ```bash
  docker build -t ml-serve:v1 .
  ```

- Verify the results

  Verify the image exists

  ```bash
  docker images ml-serve:v1
  ```

  Run the container

  ```bash
  docker run --rm -p 8090:8080 ml-serve:v1
  ```

  Open a new terminal and test the `/health` endpoint

  ```bash
  curl http://localhost:8090/health
  ```

  Should return `{"status": "ok"}`

- Stop the running container
