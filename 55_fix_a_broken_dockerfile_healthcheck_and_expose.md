### Task

The xFusionCorp Industries ML platform team deploys Flask-based inference APIs as Docker images, incorporating Docker-native `HEALTHCHECK` instructions. This allows operators to easily verify the serving status of the process by running `docker inspect --format='{{.State.Health.Status}}'`. However, the draft `Dockerfile` located at `/root/code/ml-health/` currently does not meet these standards; executing `docker inspect --format='{{.State.Health.Status}}' ml-health-api` returns `unhealthy`, and `docker inspect --format '{{.Config.ExposedPorts}}' ml-health:v1` shows no exposed ports. Your task is to rectify the `HEALTHCHECK` target and include the missing `EXPOSE` declaration.

1. The Docker daemon is already running. `docker version` can be run in a VS Code terminal to confirm. With the current image built and run as `ml-health-api`, `docker inspect --format='{{.State.Health.Status}}' ml-health-api` reports `unhealthy` and `docker inspect --format '{{.Config.ExposedPorts}}' ml-health:v1` shows no exposed ports.

2. The project layout under `/root/code/ml-health/`:
   - `app.py` – Flask API serving `GET /health` (returns `{"status": "ok"}` / `200`) and `POST /predict` (returns a rule-based fraud flag) on port `8085`. Correct.
   - `Dockerfile` – `python:3.11-slim`, installs flask, copies `app.py`, carries a `HEALTHCHECK` + `CMD`. The corrected image is built as `ml-health:v1` and run as a container named `ml-health-api` with host port `8085` published.

3. The end state must include:
   - `docker inspect --format '{{.Config.ExposedPorts}}' ml-health:v1` reports `map[8085/tcp:{}]`.
   - After `docker run`, `docker inspect --format='{{.State.Health.Status}}' ml-health-api` reads `healthy` within ~15 seconds.
   - `curl http://localhost:8085/health` returns `{"status": "ok"}` with HTTP 200.

`HEALTHCHECK` reruns its `CMD` every `--interval` seconds and flips the state to `unhealthy` after `--retries` consecutive failures. `EXPOSE` does not change networking (that is done by `-p`)—it writes image metadata so `docker inspect` and downstream orchestrators know which port the image intends to serve on.

### Solution

- Change directory

  ```bash
  cd ml-health
  ```

- Update the Dockerfile

  ```docker
  FROM python:3.11-slim

  WORKDIR /app

  RUN pip install --no-cache-dir flask

  COPY app.py /app/app.py

  EXPOSE 8085

  HEALTHCHECK --interval=5s --timeout=3s --start-period=3s --retries=3 \
    CMD python3 -c "import urllib.request; urllib.request.urlopen('http://localhost:8085/health')" || exit 1

  CMD ["python3", "/app/app.py"]
  ```

- Build and run the container

  ```bash
  docker build -t ml-health:v1 .
  docker rm ml-health-api
  docker run --name ml-health-api -p 8085:8085 -d ml-health:v1
  ```

- Verify the result

  The following should return `map[8085/tcp:{}]`.

  ```bash
  docker inspect --format='{{.Config.ExposedPorts}}' ml-health:v1
  ```

  The following should return `healthy`

  ```bash
  docker inspect --format='{{.State.Health.Status}}' ml-health-api
  ```

  The following should return `{"status":"ok"}`

  ```bash
  curl http://localhost:8085/health
  ```
