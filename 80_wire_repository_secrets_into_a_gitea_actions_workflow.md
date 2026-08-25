### Task

The xFusionCorp Industries ML platform team requires that every pull request (PR) in the `fraud-detector` repository logs a training run to MLflow and registers the resulting model version. However, they prefer not to include the tracking URL or an API token in plaintext within the workflow file. Gitea and a local MLflow 3.x server are currently operational, and a teammate has initiated a PR titled **Register trained model on every push**. Currently, the first run fails because the `register` job is attempting to access `MLFLOW_TRACKING_URI` and `MLFLOW_TOKEN` from the environment, but these variables are not populated. Your task is to provision these two values as repository secrets and integrate them into the workflow to ensure the run succeeds.

1. The Gitea UI is on port `3000` (**Gitea** button); admin credentials `gitea-admin` / `gitea2026`. The MLflow UI is on port `5000` (**MLflow UI** button). The repo is at `http://localhost:3000/gitea-admin/fraud-detector` and a working clone is at `/root/code/fraud-detector`, already checked out on branch `add-registry-push`. The PR is pre-opened.

2. The shipped `.gitea/workflows/ci.yml` declares a `register` job that runs `python3 -m src.register`. `src/register.py` reads `MLFLOW_TRACKING_URI` and `MLFLOW_TOKEN` from `os.environ` and exits non-zero if either is missing, so on the first run the job fails with that error. Two pieces are needed: the repository secrets `MLFLOW_TRACKING_URI` (value `http://localhost:5000`) and `MLFLOW_TOKEN` (any non-empty string, e.g. `fraud-detector-ci-token` — the lab's MLflow does not enforce auth, but the script refuses to run without the value so a missing secret surfaces as a clear failure), created under the repo's **Settings → Actions → Secrets**; and the workflow wiring that exports each secret into the `register` job's environment under the same name.

3. The end state must include:
   - `GET /api/v1/repos/gitea-admin/fraud-detector/actions/secrets` lists both `MLFLOW_TRACKING_URI` and `MLFLOW_TOKEN`.
   - The `register` job in the workflow references each secret via `${{ secrets.<NAME> }}` inside an `env:` block (job-level or step-level).
   - The PR head commit's combined status is `success`.
   - MLflow's registered-model endpoint reports `fraud-detector` with at least one version.

Repository secrets are the CI version of an environment-specific config file. The YAML stays identical between dev, staging, and prod; only the secret values change. This is also the pattern you extend when you add a PyPI token, an S3 access key, or a Kubernetes kubeconfig to a workflow—never paste the value into the committed file.

### Solution

- Add repo secrets

  ```
  Gitea UI -> Repo -> Settings -> Actions -> Secrets -> Add Secret
  ```

  <img src="./assets/assets_80/add_secret_1.png" alt="Add secret 1" />

  <br />

  <img src="./assets/assets_80/add_secret_2.png" alt="Add secret 2" />

- Update `.gitea/workflows/ci.yml`

  ```yml
  name: CI

  on:
    pull_request:
      branches: [main]
    push:
      branches: [main]

  jobs:
    lint:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - name: Install ruff
          run: pip install --break-system-packages ruff
        - name: Run ruff
          run: ruff check src tests

    test:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - name: Install pytest + runtime deps
          run: pip install --break-system-packages pytest pandas numpy scikit-learn joblib
        - name: Run all tests
          run: python3 -m pytest tests -v

    register:
      runs-on: ubuntu-latest
      env:
        MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
        MLFLOW_TOKEN: ${{ secrets.MLFLOW_TOKEN }}
      steps:
        - uses: actions/checkout@v4
        - name: Install registration deps
          run: pip install --break-system-packages mlflow numpy scikit-learn joblib pandas
        - name: Train and register
          run: python3 -m src.register
  ```

- Commit and push the changes

  ```bash
  cd fraud-detector/
  git add .gitea/workflows/ci.yml
  git commit -m "configure mlflow secrets for mode registration"
  git push
  ```

- Verify

  The following lists the secrets

  ```bash
  curl -u gitea-admin:gitea2026 http://localhost:3000/api/v1/repos/gitea-admin/fraud-detector/actions/secrets
  ```

  The PR head commit's combined status is success

  MLflow's registered-model endpoint reports fraud-detector with at least one version.
