### Task

The xFusionCorp Industries ML platform team has implemented a fraud-detection image and stored it in a private Docker registry for downstream clusters to access by tag. The registry is currently operational on host port `5555`, and there exists a `push.sh` script located at `/root/code/ml-registry/` that builds the image. However, the script does not yet include the steps necessary for publishing the image to the registry, which is marked as a `TODO`. Your objective is to complete the publishing flow within the script. Specifically, ensure that when you run `./push.sh`, it successfully builds the image labeled `fraud-detector:v1`, tags it appropriately for the local registry, pushes the image, and confirms that the registry's HTTP catalogue responds with `{"repositories":["fraud-detector"]}`.

1. The Docker daemon is already running and a `registry:2` container named `local-registry` is already up on host port `5555` (→ container port 5000).

2. The project layout under `/root/code/ml-registry/`:
   - `train.py` – Fits a tiny RandomForest and writes `/app/model.pkl`. Correct.
   - `Dockerfile` – `python:3.11-slim` base, installs `sklearn` + `numpy` + `joblib`, runs `train.py` at build time so the model is baked into the image. Correct.
   - `push.sh` – Executable shell script that `docker build`s `fraud-detector:v1`; the registry publish flow (naming the image for the registry and pushing it) is left as a `TODO` to author.

3. The end state must include:
   - `docker images fraud-detector:v1` lists the locally-built image.
   - `curl http://localhost:5555/v2/\_catalog` returns a JSON body with `fraud-detector` in the repositories array.
   - `curl http://localhost:5555/v2/fraud-detector/tags/list` returns a JSON body carrying `v1` in the `tags` array.

`docker tag` only writes local metadata; nothing reaches the registry until `docker push` runs.

### Solution

- Change directory

  ```bash
  cd ml-registry
  ```

- Update the `push.sh`

  ```bash
  #!/bin/bash
  # Build the fraud-detector image and publish it to the local private
  # registry so downstream clusters can pull it by tag.
  #
  # Run from /root/code/ml-registry/.
  set -euo pipefail

  cd "$(dirname "$0")"

  IMAGE="fraud-detector:v1"

  docker build -t "$IMAGE" .

  # TODO: publish "$IMAGE" to the lab's private registry so downstream
  #       clusters can pull it. The registry runs on host port 5555
  #       (registry:2 -> container port 5000). Author the publish flow
  #       below so that, after ./push.sh runs, the registry catalogue at
  #       http://localhost:5555/v2/_catalog lists "fraud-detector" and
  #       its tags/list carries "v1". Remember an image only reaches a
  #       registry once it is both named for that registry and pushed —
  #       naming it alone writes local metadata only.

  REGISTRY="localhost:5555"

  docker tag "$IMAGE" "$REGISTRY/$IMAGE"

  docker push "$REGISTRY/$IMAGE"
  ```

- Run the script

  ```bash
  ./push.sh
  ```

- Verify the end state

  ```bash
  docker images fraud-detector:v1
  curl http://localhost:5555/v2/_catalogurl
  http://localhost:5555/v2/fraud-detector/tags/list
  ```

Note:

- The lab explicitly requires `docker tag` instead of `docker image tag`
