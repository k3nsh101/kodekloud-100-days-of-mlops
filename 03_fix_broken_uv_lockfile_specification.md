### Task

The xFusionCorp Industries ML team uses `uv` and lockfiles to keep Python dependencies reproducible across machines. A teammate has left behind a `requirements.in` specification that does not match the team's standard. Correct it and compile it into a pinned lockfile.

1. A high-level dependency specification exists at `/root/code/fraud-detection/requirements.in`. `uv` is already installed.

2. The corrected specification must meet the following requirements:
   - it lists exactly these four top-level packages: `scikit-learn`, `mlflow`, `pandas`, and `numpy`;
   - every package carries a version constraint that `uv` can actually satisfy against PyPI.

3. Review the existing `requirements.in`, and correct everything that does not match the requirements above.

4. From the project directory, compile the corrected specification into a pinned lockfile:

   ```bash
   uv pip compile requirements.in -o requirements.txt
   ```

5. The resulting `requirements.txt` must pin each of the four top-level packages to an exact version using `==`, and must also include the transitive dependencies that `uv` resolved.

### Solution

Update the `requirements.in`

```
# Fraud detection project dependencies
scikit-learn
mlflow>=3.12
numpy
pandas
```

Then run

```bash
uv pip compile requirements.in -o requirements.txt
```
