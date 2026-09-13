### Task

The xFusionCorp Industries ML platform team aims to train multiple model variants simultaneously and select the best performing model. The `train-parallel-variants` WorkflowTemplate has been created but is incomplete: the `train-variant` and `pick-best` step templates are defined, yet the `main` template lacks its `withParam` fan-out and the `pick-best` fan-in step. Your task is to complete both components: fan out the `train-variant` step over the `estimators_list` parameter using `withParam`, and incorporate the `pick-best` reducer as a second step. After applying the template, submit it twice through the Argo UI: first, with a deliberately faulty entry (ensuring one branch fails), and then with a valid list (confirming both the fan-out and reducer complete successfully).

1. The scaffolded template is at `/root/code/argo/train-parallel-variants.yaml`. It is NOT applied yet — the Argo UI's **Workflow Templates** list (port `5000`) stays empty until the template is applied to the `argo` namespace. The `train-variant` template validates its `n_estimators` input as a positive integer and exits 1 otherwise.

2. Two `TODO`-marked pieces are missing from the main template:
   - **TODO 1 — fan-out**: the `train` step runs only once instead of fanning out over the `estimators_list` parameter, so it never becomes the N parallel pods the sweep needs.
   - **TODO 2 — fan-in**: the reducer step group that runs after the parallel branches finish is absent, so nothing picks the best result once the variants complete.

3. With the template applied and appearing in the UI, submitting it twice demonstrates the fan-out and its failure mode: an `estimators_list` with one obviously-bad entry alongside valid positive integers red-lights the bad branch while the others go green, leaves `pick-best` Omitted, and finishes `Failed`; a clean list of three valid positive integers turns every `train-variant` branch green, runs `pick-best`, and finishes `Succeeded`.

4. The end state must include:
   - `GET /api/v1/workflow-templates/argo/train-parallel-variants` returns the template, whose `main` step fans out over `estimators_list` with `withParam` and includes a `pick-best` fan-in step.
   - At least two workflows exist with `spec.workflowTemplateRef.name == train-parallel-variants`.
   - The most recent workflow's `status.phase == Succeeded`, with ≥3 `train-variant` Pod nodes all `Succeeded` plus one `pick-best` node Succeeded.

`withParam` is Argo's fan-out primitive—one step definition, N parallel pods, one template per input value. Because every item runs independently, one bad value does not stop the others; it only blocks the fan-in reducer (`pick-best`) from receiving a complete set. That isolation is both the pattern's value (a 99-of-100 sweep still gives you 99 models) and its failure mode (one bad row in the input list red-lights the release).

### Solution

- Go to Argo UI

- Create new workflow template with following manifest.

  ```
  apiVersion: argoproj.io/v1alpha1
  kind: WorkflowTemplate
  metadata:
    name: train-parallel-variants
    namespace: argo
  spec:
    entrypoint: main
    arguments:
      parameters:
        - name: estimators_list
          value: '["10","50","100"]'
    templates:
      - name: main
        steps:
          # TODO 1: Fan out `train-variant` over the estimators_list
          # parameter. On this step add `withParam:` referencing
          # workflow.parameters.estimators_list, and pass each list item to
          # the template as its n_estimators input via arguments.parameters
          # (value "{{item}}"). Without withParam this runs a single
          # train-variant with no input and fails the required-parameter
          # check.
          - - name: train
              template: train-variant
              withParam: "{{workflow.parameters.estimators_list}}"
              arguments:
                parameters:
                  - name: n_estimators
                    value: "{{item}}"
          # TODO 2: Add the fan-in reducer as a SECOND step group so it runs
          # after the parallel fan-out completes -- a step named `pick-best`
          # that calls the `pick-best` template below.
          - - name: pick-best
              template: pick-best
      - name: train-variant
        inputs:
          parameters:
            - name: n_estimators
        script:
          image: alpine:3.19
          command: [sh]
          source: |
            N='{{inputs.parameters.n_estimators}}'
            case "$N" in
              ''|*[!0-9]*)
                echo "[train] ERROR: n_estimators must be a positive integer, got: $N"
                exit 1
                ;;
            esac
            echo "[train] fitting with n_estimators=$N"
            sleep 2
            echo "model-n$N"
      - name: pick-best
        container:
          image: alpine:3.19
          command: [sh, -c]
          args: ["echo '[pick_best] selected: model-n100'"]

  ```

- Submit the template with bad entry for parameter list

  ```
  [10, 50, 100, "BAD"]
  ```

- Submit the template with correct entries.

  ```
  [10, 50, 100]
  ```

- Verify

  The following should return the template, whose `main` step fans out over `estimators_list` with `withParam` and includes a `pick-best` fan-in step.

  ```bash
  curl http://localhost:5000/api/v1/workflow-templates/argo/train-parallel-variants
  ```

  And the other requirements in the last point should be satisfied.
