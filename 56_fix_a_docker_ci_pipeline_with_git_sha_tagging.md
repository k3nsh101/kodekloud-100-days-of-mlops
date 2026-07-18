### Task

The xFusionCorp Industries ML platform team operates a shell-based Docker CI pipeline for the fraud-detection Flask service. In this process, tests are executed, the image is built, a short git SHA is applied as the tag, and the tagged image is subsequently pushed to the local private registry. However, the pre-staged `build.sh` located at `/root/code/ci/` does not currently execute cleanly from start to finish. Your objective is to rectify the configuration so that `./build.sh` completes its execution without errors and that the registry catalog displays `ml-ci-app` tagged with the current git short SHA.

1. The Docker daemon is already running and a `registry:2` container named `local-registry` is already up on host port `5555`.

2. The repository layout under `/root/code/ci/`:
   - `app/app.py` – Flask service exposing `/health` + `/predict` on port `8086`. Correct.
   - `app/test_app.py` – Three pytest cases covering `/health`, the fraud-flag flow, and the pass-through flow. Correct.
   - `app/Dockerfile` – `python:3.11-slim`, installs flask, COPYs `app.py`, exposes 8086, runs the Flask app. Correct.
   - `app/.git/` – A local git repository initialised at startup with a single "Initial CI baseline" commit. Correct.
   - `build.sh` – Executable shell script with four stages (test → build → tag → push). Needs attention.

3. The end state must include:
   - `./build.sh` runs end-to-end without non-zero exit.
   - `docker images ml-ci-app:latest` lists the locally-built image.
   - `curl http://localhost:5555/v2/_catalog` lists `ml-ci-app` in the `repositories` array.
   - `curl http://localhost:5555/v2/ml-ci-app/tags/list` lists the current `git -C app rev-parse --short HEAD` value in the `tags` array.

Run `./build.sh` against the scaffold as-is; each re-run surfaces the next blocker. All fixes live inside `build.sh`.

### Solution

- Change directory

  ```bash
  cd /root/code/ci
  ```

- Update the `./build.sh`

  Change the REGISTRY port, test directory and GIT_SHA variable name. The lab expects `SHA` instead of `GIT_SHA`

  ```bash
  #!/bin/bash
  # Shell-based CI pipeline for the ml-ci-app image.
  #
  # Stages: test -> build -> tag (git SHA) -> push to local registry.
  # Run from /root/code/ci/.
  set -euo pipefail

  cd "$(dirname "$0")"

  IMAGE="ml-ci-app"
  REGISTRY="localhost:5555"

  # --- Stage 1: test
  echo "[ci] stage 1/4 — running tests"
  python3 -m pytest app/

  # --- Stage 2: build
  echo "[ci] stage 2/4 — building image"
  docker build -t "$IMAGE:latest" app/

  # --- Stage 3: tag with short git SHA
  echo "[ci] stage 3/4 — tagging"
  SHA=$(git -C app rev-parse --short HEAD)
  TAGGED="$REGISTRY/$IMAGE:$SHA"
  docker tag "$IMAGE:latest" "$TAGGED"

  # --- Stage 4: push
  echo "[ci] stage 4/4 — pushing"
  docker push "$TAGGED"

  echo "[ci] complete: $TAGGED"
  ```

- Run the script

  ```bash
  ./build.sh
  ```

- Verify resutls

  ```bash
  docker images ml-ci-app:latest
  ```

  The following shows `{"repositories":["ml-ci-app"]}`

  ```bash
  curl http://localhost:5555/v2/_catalog
  ```

  The following shows `{"name":"ml-ci-app","tags":["feb4c3a"]}`. The tag will be different.

  ```bash
  curl http://localhost:5555/v2/ml-ci-app/tags/list
  ```
