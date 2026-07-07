### Task

The xFusionCorp Industries ML platform team wants data-schema contracts on every batch that feeds the fraud-detector model—catch the malformed row upstream of training, not three hours later in production. A Great Expectations project is already initialised at `/root/code/dataquality/gx/` with a pandas data source reading `data/transactions.csv`, an empty `fraud_schema` suite, and a `default` checkpoint wired to publish results to Data Docs on every run. Your task is to populate the suite with four expectations and run the checkpoint so Data Docs shows them green.

1. Open `/root/code/dataquality/author_expectations.py` in the VS Code editor. Four numbered TODOs call out the expectations the suite must carry:
   - TODO 1: `ExpectTableColumnsToMatchSet` with `column_set=["amount", "hour", "num_tx_past_day", "is_fraud"]`.
   - TODO 2: `ExpectColumnValuesToBeBetween` on `amount` with `min_value=0` (no negative amounts).
   - TODO 3: `ExpectColumnValuesToBeBetween` on `hour` with `min_value=0` and `max_value=23`.
   - TODO 4: `ExpectColumnValuesToBeInSet` on `is_fraud` with `value_set=[0, 1]`.

   The helpful imports are already in place at the top of the file (`import great_expectations as gx` and `import great_expectations.expectations as ge`).

2. Run the script:

   ```python
   python3 /root/code/dataquality/author_expectations.py
   ```

   The script persists the suite to disk (`gx/expectations/fraud_schema.json`) and then executes the `default` checkpoint, which validates `transactions.csv` against the new suite and refreshes the Data Docs site.

3. Click the **Data Docs** button at the top of the lab (port `8081`). The landing page lists one validation run under `fraud_schema`—click through to the run's detail page and confirm every expectation's pill reads green (`Success`).

4. The end state must include:
   - `gx/expectations/fraud_schema.json` has all four expectations by type (`expect_table_columns_to_match_set`, two `expect_column_values_to_be_between entries` – One per column — and `expect_column_values_to_be_in_set`).
   - Each expectation's kwargs match the TODO spec above.
   - The most recent validation JSON under `gx/uncommitted/validations/` has `success: true`.
   - The Data Docs index page served on `:8081` references `fraud_schema`.

Great Expectations treats data quality as code—expectation suites are versioned artefacts in the same repo as the model that consumes the data, run by the same CI that runs pytest. A run's result JSON is machine-readable (a downstream CI-gate lab consumes it), and Data Docs is the human-readable rendering of the same content. This lab lays the ground for both.

### Solution

- Update `/root/code/dataquality/author_expectations.py`

  ```python
  """Author the fraud_schema expectation suite and run the default checkpoint.

  Startup has already initialised the Great Expectations project,
  registered a pandas data source over ``data/transactions.csv``,
  created an empty ``fraud_schema`` suite, and wired it into a
  ``default`` checkpoint that refreshes Data Docs on every run. The
  only missing piece is the expectations themselves -- add them in
  the TODO block below, then execute the file.
  """
  from __future__ import annotations

  import great_expectations as gx
  import great_expectations.expectations as ge

  PROJECT_ROOT = "/root/code/dataquality"   # GE creates <root>/gx/ as home
  SUITE_NAME = "fraud_schema"
  CHECKPOINT_NAME = "default"


  def main() -> None:
      context = gx.get_context(mode="file", project_root_dir=PROJECT_ROOT)

      # Self-healing: if the startup scaffold skipped the suite for any
      # reason, add_or_update still returns a fresh empty one.
      suite = context.suites.add_or_update(
          gx.ExpectationSuite(name=SUITE_NAME),
      )
      suite.expectations = []  # start clean on every authoring pass

      # ------------------------------------------------------------------
      # TODO 1: Declare the required schema -- the four columns
      #         amount, hour, num_tx_past_day, is_fraud must all be
      #         present on every batch.
      #
      #         suite.add_expectation(
      #             ge.ExpectTableColumnsToMatchSet(
      #                 column_set=["amount", "hour", "num_tx_past_day", "is_fraud"],
      #             )
      #         )
      #
      # TODO 2: Guard `amount` -- no negative transactions.
      #         Use ge.ExpectColumnValuesToBeBetween with min_value=0.
      #
      # TODO 3: Guard `hour` -- valid 24-hour range.
      #         Use ge.ExpectColumnValuesToBeBetween with min_value=0
      #         and max_value=23.
      #
      # TODO 4: Guard `is_fraud` -- binary label.
      #         Use ge.ExpectColumnValuesToBeInSet with value_set=[0, 1].
      # ------------------------------------------------------------------

      # (expectations go here)
      suite.add_expectation(
          ge.ExpectTableColumnsToMatchSet(
              column_set = [
                  "amount",
                  "hour",
                  "num_tx_past_day",
                  "is_fraud"
              ]
          )
      )

      suite.add_expectation(
          ge.ExpectColumnValuesToBeBetween(
              column="amount",
              min_value=0
          )
      )

      suite.add_expectation(
          ge.ExpectColumnValuesToBeBetween(
              column="hour",
              min_value=0,
              max_value=23
          )
      )

      suite.add_expectation(
          ge.ExpectColumnValuesToBeInSet(
              column="is_fraud",
              value_set=[0, 1]
          )
      )

      context.suites.add_or_update(suite)
      print(f"Persisted {len(suite.expectations)} expectations to `{SUITE_NAME}`")

      checkpoint = context.checkpoints.get(name=CHECKPOINT_NAME)
      result = checkpoint.run()
      print(f"Checkpoint `{CHECKPOINT_NAME}` result: success={result.success}")


  if __name__ == "__main__":
      main()
  ```

- Run the script

  ```python
  python3 /root/code/dataquality/author_expectations.py
  ```

- Verify the end state
