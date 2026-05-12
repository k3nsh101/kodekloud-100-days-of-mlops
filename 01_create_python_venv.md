### Task

The xFusionCorp Industries data science team needs a standardised Python environment for their new ML project. Set up a virtual environment with the required ML libraries on the `controlplane` host.

1. Create a Python virtual environment named `ml-env` under `/root/code/` using `python3 -m venv`.
2. Activate the environment and install the following packages: `numpy`, `pandas`, `scikit-learn`, and `matplotlib`.
3. Generate a `requirements.txt` file using `pip freeze` and save it at `/root/code/requirements.txt`.

### Solution

```bash
python3 -m venv ml-env

source ml-env/bin/activate

pip install numpy pandas scikit-learn matplotlib

pip freeze > /root/code/requirements.txt
```
