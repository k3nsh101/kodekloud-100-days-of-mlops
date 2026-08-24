### Task

The xFusionCorp Industries ML platform team's Continuous Integration (CI) system currently trains the model and generates a confusion matrix in PNG format. However, these artifacts are discarded when the runner tears down the workspace, preventing reviewers on a Pull Request (PR) from accessing them. A teammate has submitted a PR titled **Publish Training Artifacts from CI** in the `fraud-detector` repository. Your task is to integrate `actions/upload-artifact` into the existing `report` job, ensuring that `metrics.json` and `confusion_matrix.png` are available as a downloadable zip file on the run page.

1. The Gitea UI is running on port `3000` (the `Gitea` button opens the login page). Admin credentials: `gitea-admin` / `gitea2026`. The repo lives at `http://localhost:3000/gitea-admin/fraud-detector` and a working clone is at `/root/code/fraud-detector`, already checked out on branch `add-artifact-upload`. The PR is pre-opened.

2. The current `report` job in `.gitea/workflows/ci.yml` already installs `numpy scikit-learn joblib matplotlib`, runs `python3 -m src.train` (writes `artifacts/model.joblib` and `artifacts/metrics.json`), runs `python3 -m src.plot` (writes `artifacts/confusion_matrix.png`), and ends with `ls -la artifacts/` — but the produced files are discarded when the run finishes. The job needs a final step that uploads the `artifacts/` directory as a named artefact (`model-report`) using `actions/upload-artifact`. Use `@v3` — Gitea's runner rejects `@v4`.

3. The end state must include:
   - The `report` job contains a step that uses `actions/upload-artifact@\*` with a non-empty `path` and the artefact name `model-report`.
   - The run-level artefact download at `/gitea-admin/fraud-detector/actions/runs/<id>/artifacts/model-report` returns a zip containing both `metrics.json` and `confusion_matrix.png` by basename.
   - The PR head commit's combined status is `success`.

Artefacts are CI's answer to the question 'what did that run actually produce?'. A green check-mark tells you the code compiled; an uploaded artefact tells you what shipped out the other end. For ML runs, this is where the metrics JSON, the confusion matrix plot, and the pickled model live before anyone promotes them to a registry.

### Solution

- Change directory

  ```bash
  cd fraud-detector/
  ```

- Update the `.gitea/workflows/ci.yml`

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

    report:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - name: Install report deps
          run: pip install --break-system-packages numpy scikit-learn joblib matplotlib
        - name: Train
          run: python3 -m src.train
        - name: Plot confusion matrix
          run: python3 -m src.plot
        - name: List produced artefacts
          run: ls -la artifacts/
        - name: Upload artefacts
          uses: actions/upload-artifact@v3
          with:
            name: model-report
            path: artifacts/
  ```

- Commit and push the changes

  ```bash
  git add .gitea/workflows/ci.yml
  git commit -m "ci: add upload artefacts"
  git push
  ```

- Verify the results

  The run-level artefact download at /gitea-admin/fraud-detector/actions/runs/<id>/artifacts/model-report returns a zip containing both metrics.json and confusion_matrix.png by basename.

  The PR head commit's combined status is success.
