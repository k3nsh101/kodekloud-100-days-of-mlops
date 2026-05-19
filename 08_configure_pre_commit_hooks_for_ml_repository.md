### Task

The xFusionCorp Industries ML team enforces code quality on every commit via `pre-commit`. A draft `.pre-commit-config.yaml` exists in the git repository at `/root/code/fraud-detection/`, but it does not match the team's standard and `pre-commit run --all-files` fails against it. Correct the configuration.

1. A git repository already exists at `/root/code/fraud-detection/` with `.pre-commit-config.yaml` and `process.py` already tracked. `pre-commit` is installed system-wide.
2. The corrected configuration must declare the following five hooks so that `pre-commit run --all-files` executes every one of them:
   - `trailing-whitespace`, `end-of-file-fixer`, and `check-yaml` – All three sourced from the `pre-commit/pre-commit-hooks` repository, pinned to a current release;
   - `ruff` – Sourced from the `astral-sh/ruff-pre-commit` repository, pinned to a current release;
   - `black` – Sourced from the `psf/black-pre-commit-mirror` repository, pinned to a current release.

3. Every repository entry in the configuration must include a `rev:` field.

4. Review the existing `.pre-commit-config.yaml` and correct everything that prevents the hooks above from running.

5. Once the configuration is correct, register the hooks with git and run them against the tracked files:

   ```bash
   pre-commit install
   pre-commit run --all-files
   ```

Tip: `pre-commit autoupdate` queries each referenced repository and rewrites the `rev:` pins to the latest released tag. This is the standard way to discover current versions without looking them up by hand.

### Solution

- Change directory to the src file

  ```bash
  cd fraud-detection
  ```

- Find the current versions of the hooks and update the details of `.pre-commit-config.yaml` as follows
  Use `pre-commit autoupdate` to rewrite the `rev:` pins to the latest released tag automatically.

  ```
  repos:
    - repo: https://github.com/pre-commit/pre-commit-hooks
      rev: v6.0.0
      hooks:
        - id: trailing-whitespace
        - id: end-of-file-fixer
        - id: check-yaml

    - repo: https://github.com/astral-sh/ruff-pre-commit
      rev: v0.15.13
      hooks:
        - id: ruff-check

    - repo: https://github.com/psf/black-pre-commit-mirror
      rev: 26.5.1
      hooks:
        - id: black
  ```

- The lab expects the ruff hook id to be `ruff` (not `ruff-check` as specified in the official docs)

- Register the hooks and run them against the tracked files

  ```bash
  pre-commit install
  pre-commit run --all-files
  ```
