### Task

A teammate has added the transactions dataset to the xFusionCorp Industries fraud-detection repository, but it was committed directly to Git instead of being tracked with DVC. Bring the repository in line with the team standard—every dataset under `data/` must be tracked by DVC, not by Git.

1. A project exists at `/root/code/fraud-detection/` with DVC already initialised. The dataset `data/raw/transactions.csv` is currently tracked by Git, and the team standard requires DVC to own it instead.

2. Stop Git from tracking the dataset without deleting it from disk.

3. Track the same dataset with DVC so a `.dvc` pointer file is produced and `data/raw/.gitignore` excludes the dataset itself.

4. Stage the new `.dvc` pointer and the new `.gitignore`, then record a Git commit with the message `Track transactions dataset with DVC`.

Once tracking is moved to DVC, the **DVC TRACKED** section in the EXPLORER panel will list the dataset, confirming the extension recognises it as a DVC-managed file.

### Solution

- Add a `.gitignore` file under `data/raw/` with the following content

  ```
  transactions.csv
  ```

- Change directory

  ```bash
  cd fraud-detection
  ```

- Stop git from tracking the dataset without deleting it from disk.

  ```bash
  git rm --cached data/raw/transactions.csv
  ```

  `--cached` ensure that the file is not deleted from the disk.

- Track the dataset with DVC.

  ```bash
  dvc add data/raw/transactions.csv
  ```

- Stage and commit changes to git

  ```bash
  git add .
  git commit -m "Track transactions dataset with DVC"
  ```
