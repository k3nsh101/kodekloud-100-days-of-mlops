### Task

The xFusionCorp Industries ML platform team keeps the fraud-detection feature definitions in a Feast repository at `/root/code/fraud-detection/feature_repo/`. A draft `features.py` exists there, and the registry that `feast apply` wrote is inconsistent with the source data in `data/transactions.parquet`. Your task is to correct `features.py`, re-apply the registry, and confirm the corrected `customer_transaction_features` view in the Feast UI.

1. The Feast UI is already running on port `8888`. The **Feast UI** button at the top of the lab can be opened to confirm—the dashboard loads the `fraud_detection` project with one entity and one feature view carrying the draft declarations.

2. The repository layout under `/root/code/fraud-detection/feature_repo/`:
   - `feature_store.yaml` – The Feast config (project `fraud_detection`, local provider, sqlite online store, file offline store). Correct and must remain intact.
   - `data/transactions.parquet` – A 200-row synthetic source keyed by `customer_id`; carries `amount` as Float32, `hour` + `num_tx_past_day` + `is_fraud` as Int64, and an `event_timestamp` column. Correct and must remain intact.
   - `features.py` – Declares one `FileSource`, one `Entity`, and one `FeatureView`. Needs correction to match the source.
   - `data/registry.db` – Written by `feast apply` at startup from the draft definitions; must be re-applied after the fixes.

3. Open `features.py` in the VS Code editor, align the declarations with the source, save, and run `feast apply` from inside `/root/code/fraud-detection/feature_repo/`.

4. The end state must include:
   - The `customer` entity in the registry has `join_keys = ["customer_id"]`.
   - The `customer_transaction_features` feature view's `amount` field is declared as `Float32` (matching the parquet writer's output type).
   - `feast apply` exits without error and the Feast UI reflects the corrected entity and feature-view schema.

The Feast UI's Entities and Feature Views tabs surface the applied values directly—the current (draft) values are visible there so the required change is easy to eyeball against the task's end-state.

### Solution

- Change directory

  ```bash
  cd fraud_detection/feature_repo
  ```

- Update `features.py`

  Update `join_keys` and `dtype` of `amount` field

  ```python
  """Feature definitions for the fraud-detection project.

  Declares the `customer` entity, the `transactions` batch source, and
  one feature view (`customer_transaction_features`) that exposes the
  customer's `amount`, `hour`, and `num_tx_past_day` features to the
  feature store.

  Every non-end-state concern is correctly wired — imports, source
  pointer, feature-view name, entity-to-view binding, TTL. Adjust
  the entity `join_keys` so it maps onto the column Feast actually
  needs to look up in the source, and adjust the `amount` feature's
  declared `dtype` so it matches the type written by the generator.
  """
  from datetime import timedelta

  from feast import Entity, FeatureView, Field, FileSource
  from feast.types import Float32, Int64, String

  transactions_source = FileSource(
      path="data/transactions.parquet",
      timestamp_field="event_timestamp",
  )

  customer = Entity(
      name="customer",
      join_keys=["customer_id"],
      description="Customer identifier keyed by the transactions source.",
  )

  customer_transaction_features = FeatureView(
      name="customer_transaction_features",
      entities=[customer],
      ttl=timedelta(days=365),
      schema=[
          Field(name="amount", dtype=Float32),
          Field(name="hour", dtype=Int64),
          Field(name="num_tx_past_day", dtype=Int64),
      ],
      source=transactions_source,
      online=True,
  )
  ```

- Register feature definitions and deploy the feature store

  ```bash
  feast apply
  ```
