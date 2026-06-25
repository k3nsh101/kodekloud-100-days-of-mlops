### Task

The xFusionCorp Industries ML platform team ships fraud-detection models built on a tiny PyTorch network. The training script needs to run on whichever accelerator the host exposes—CUDA GPUs in the production cluster, plain CPU on this lab's nodes—so the same code works on every target. A draft trainer exists at `/root/code/fraud-detection/src/models/train_pytorch.py`, but it assumes CUDA is always present and crashes on the first tensor move, and the device parameter it logs to MLflow is hardcoded. Your task is to make the trainer device-aware and ensure the MLflow run truthfully reports the device it actually used.

1. The MLflow tracking server is already running on port `5000`. The **MLflow UI** button at the top of the lab can be opened to confirm—the dashboard loads with an empty `gpu-training` experiment. PyTorch (CPU build) is baked into the lab image; `import torch` works out of the box. The host does not expose a GPU (`torch.cuda.is_available()` returns `False`).

2. The project layout under `/root/code/fraud-detection/`:
   - `data/train.csv` – The same 200-row synthetic binary-classification dataset the rest of the Training section uses.
   - `src/models/train_pytorch.py` – The trainer scaffold. The two-layer feedforward network, the optimiser, the loss function, the MLflow experiment setup, and the model-persistence call to `models/fraud_model.pt` are already wired; two specific corrections are required.

3. Open `src/models/train_pytorch.py` in the VS Code editor, correct the two problems that keep the trainer from running on this host, save, and run the script once.

4. The end state must include:
   - The script completes successfully and writes a PyTorch state-dict to `/root/code/fraud-detection/models/fraud_model.pt`.
   - One run exists in the `gpu-training` experiment on MLflow, carrying `params.device = "cpu"` and `metrics.final_loss`.
   - No bare .cuda() calls remain anywhere in train_pytorch.py.

`torch.cuda.is_available()` returns `True` on GPU hosts and `False` on CPU-only hosts. The idiomatic pattern is to build a single `device = torch.device("cuda" if torch.cuda.is_available() else "cpu")` at the top of training, use `.to(device)` on every tensor and the model, and log `str(device)` to MLflow so the run reports where it actually ran.

### Solution

- Update `/root/code/fraud-detection/src/models/train_pytorch.py`

  ```python
  """Feedforward fraud-detection trainer.

  Trains a tiny two-layer network on the synthetic transactions CSV,
  logs the run to MLflow with `params.device` + `metrics.final_loss`,
  and saves the trained weights to `models/fraud_model.pt`.

  Every non-device concern is correctly wired — data loading, model
  definition, optimizer setup, loss function, MLflow experiment. The
  current wiring assumes a CUDA GPU is always present. Adjust the
  device handling so the script runs on whichever accelerator the
  host actually exposes and the logged device parameter reflects
  the same value.
  """
  import os

  import mlflow
  import numpy as np
  import pandas as pd
  import torch
  import torch.nn as nn

  TRACKING_URI = "http://localhost:5000"
  EXPERIMENT = "gpu-training"
  TRAIN_CSV = "/root/code/fraud-detection/data/train.csv"
  MODEL_PATH = "/root/code/fraud-detection/models/fraud_model.pt"

  FEATURES = ["amount", "hour", "num_tx_past_day"]
  TARGET = "is_fraud"
  EPOCHS = 30
  LR = 0.01
  SEED = 42


  class FraudNet(nn.Module):
      def __init__(self):
          super().__init__()
          self.fc1 = nn.Linear(len(FEATURES), 8)
          self.fc2 = nn.Linear(8, 2)

      def forward(self, x):
          return self.fc2(torch.relu(self.fc1(x)))


  def main():
      torch.manual_seed(SEED)
      np.random.seed(SEED)

      mlflow.set_tracking_uri(TRACKING_URI)
      mlflow.set_experiment(EXPERIMENT)

      device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

      df = pd.read_csv(TRAIN_CSV)
      X = df[FEATURES].values.astype(np.float32)
      y = df[TARGET].values.astype(np.int64)

      X_t = torch.from_numpy(X)
      y_t = torch.from_numpy(y)

      model = FraudNet()
      model = model.to(device)

      optimizer = torch.optim.SGD(model.parameters(), lr=LR)
      loss_fn = nn.CrossEntropyLoss()

      with mlflow.start_run(run_name="fraud-mlp"):
          mlflow.log_param("device", str(device))

          xb = X_t.to(device)
          yb = y_t.to(device)

          final_loss = None
          for epoch in range(EPOCHS):
              logits = model(xb)
              loss = loss_fn(logits, yb)
              optimizer.zero_grad()
              loss.backward()
              optimizer.step()
              final_loss = float(loss.item())
              print(f"epoch {epoch:02d}  loss={final_loss:.4f}")

          mlflow.log_metric("final_loss", final_loss)

          os.makedirs(os.path.dirname(MODEL_PATH), exist_ok=True)
          torch.save(model.state_dict(), MODEL_PATH)
          print(f"Model saved to {MODEL_PATH}")


  if __name__ == "__main__":
      main()
  ```

- Run the script

  ```bash
  python3 /root/code/fraud-detection/src/models/train_pytorch.py
  ```

- Verify from the **MLflow UI** and according to the speicified requirements.
