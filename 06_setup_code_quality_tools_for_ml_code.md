### Task

The xFusionCorp Industries ML team enforces code quality with `ruff` and `black` on every pull request. The project at `/root/code/fraud-detection/` currently fails both tools. Make it pass them.

1. The project at `/root/code/fraud-detection/` contains a `pyproject.toml` and sample sources under `src/`.

2. The corrected project must meet the following requirements:
   - `ruff` and `black` are both configured with a line length of `120`.
   - `ruff` lint rule selection includes `E`, `F`, `W`, and `I`, and is declared under `[tool.ruff.lint]` – The schema required by `ruff` 0.1 and later.
   - Running `ruff check src/` from the project directory exits with status `0`.
   - Running `black --check src/` from the project directory exits with status `0`.

3. Review the existing configuration and source files, and correct everything that prevents the two commands above from exiting cleanly.

`ruff`, `black`, and `mypy` are already installed.

### Solution

- Update `pyproject.toml`

  ```
  [project]
  name = "fraud-detection"
  version = "0.1.0"

  [tool.ruff]
  line-length = 120

  [tool.ruff.lint]
  select = ["E", "F", "W", "I"]

  [tool.black]
  line-length = 120
  ```

  ```bash
  ruff check src/
  black --check src/
  ```

- Remove `import os` from `src/data/process_data.py`

- Run checks again
