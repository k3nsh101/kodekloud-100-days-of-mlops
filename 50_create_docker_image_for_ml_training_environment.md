### Task

The xFusionCorp Industries ML platform team has created a Docker image for the fraud-detection training environment, allowing every engineer to achieve consistent results by executing the command `docker run ml-trainer:v1`. A scaffold for the `Dockerfile` is located at `/root/code/ml-docker/`, with its construction outlined as numbered TODOs. Your objective is to complete the Dockerfile in accordance with the team's standards, ensuring that the command `docker build -t ml-trainer:v1 .` executes successfully and that every Python import required by the training code is correctly resolved within the image.

1. The Docker daemon is already running. `docker version` can be run in a VS Code terminal to confirm.

2. The project layout under `/root/code/ml-docker/`:
   - `train.py` – 10-row synthetic fraud-detection training stub; fits a RandomForest, prints accuracy + F1, and persists the model to `/app/model.pkl` with `joblib.dump(...)`. Correct and must remain intact.
   - `Dockerfile` – The image definition, scaffolded as five numbered TODOs (base image, working directory, dependency install, copy, command). Author each to the team standard.

3. The end state must include:
   - The base image is `python:3.11-slim`.
   - `WORKDIR /app` is set.
   - The `pip install` line installs every package the training code imports (`scikit-learn`, `pandas`, `numpy`, `joblib`).
   - `docker images ml-trainer:v1` lists the built image.
   - `docker run --rm ml-trainer:v1 python3 -c "import sklearn, pandas, numpy, joblib; print('OK')"` prints `OK`.

`docker build .` can be run repeatedly as each instruction lands; Docker re-uses cached layers so only the changed line re-runs. `train.py` is complete and must stay intact — only the `Dockerfile` is authored.

### Solution

- Change directory

  ```bash
  cd ml-docker
  ```

- Update the `Dockerfile`

  ```docker
  # ML training image for the fraud-detection trainer.
  #
  # Author each instruction below to the team standard, then build with:
  #   docker build -t ml-trainer:v1 .
  # from inside /root/code/ml-docker/.

  # TODO 1: Base image — use python:3.11-slim. Do not use an alpine
  #         base: its musl libc has no manylinux wheel for scikit-learn,
  #         so the pip install fails (or falls back to a slow source
  #         build that exceeds the lab's memory budget).
  FROM python:3.11-slim

  # TODO 2: Set the working directory to /app.
  WORKDIR /app

  # TODO 3: Install the training dependencies with pip (no cache):
  #         scikit-learn, pandas, numpy, joblib. All four are imported
  #         by train.py — joblib is used at runtime for joblib.dump, so
  #         omitting it aborts the container with ModuleNotFoundError.
  RUN pip install --no-cache-dir scikit-learn pandas numpy joblib

  # TODO 4: Copy train.py into the image at /app/train.py.
  COPY train.py .

  # TODO 5: Set the default command to run the trainer: python3 train.py.
  CMD ["python3", "train.py"]
  ```

- Build the image

  ```bash
  docker build -t ml-trainer:v1 .
  ```

- Verify the result

  The built image is shown when the following command is run

  ```bash
  docker images ml-trainer:v1
  ```

  - Test for `print OK`. The following should output `OK`

  ```bash
  docker run --rm ml-trainer:v1 python3 -c "import sklearn, pandas, numpy, joblib; print('OK')"
  ```
