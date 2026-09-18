### Task

The xFusionCorp Industries ML platform team has established KServe on their Kind cluster and deployed a `fraud-detector` InferenceService, which is backed by a PVC-mounted Scikit-learn model. However, the InferenceService is not reaching a `Ready` state, as the predictor pod remains in `Pending`. Please investigate the cause of this issue and rectify the manifest located at `/root/code/k8s/inference-service.yaml` to ensure that the InferenceService transitions to `Ready`. After making the necessary adjustments, confirm that the predictor can successfully serve a prediction.

1. KServe is installed on a kind cluster, and a `fraud-detector` InferenceService has been deployed — but it never reaches `Ready` and its predictor pod stays `Pending`. Inspect the predictor pod to see what is blocking it:

```bash
kubectl describe pod -l serving.kserve.io/inferenceservice=fraud-detector
```

2. The end state must include:
   - `kubectl get isvc fraud-detector` shows the InferenceService.
   - `spec.predictor.model.storageUri` references a PVC that exists in the namespace.
   - `.status.conditions[?(@.type=="Ready")].status == True` (tests poll up to 360 s).
   - A prediction request to the predictor's `/v1/models/fraud-detector:predict` returns a JSON `predictions` array.

KServe's `pvc://` storage scheme mounts the named PVC into the predictor pod at `/mnt/models` and lets the runtime read model artefacts from there. The reference must name a PVC that exists in the InferenceService's namespace.

### Solution

- Find the issue

  ```bash
  kubectl describe pod -l serving.kserve.io/inferenceservice=fraud-detector
  ```

  This shows that the pvc is not found.

- Change the pvc storage uri to `pvc://model-storage/`.

  ```yaml
  apiVersion: serving.kserve.io/v1beta1
  kind: InferenceService
  metadata:
    name: fraud-detector
    annotations:
      serving.kserve.io/deploymentMode: "RawDeployment"
  spec:
    predictor:
      minReplicas: 1
      maxReplicas: 1
      model:
        modelFormat:
          name: sklearn
        # storageUri bug: the real PVC is named `model-storage`.
        storageUri: "pvc://model-storage/"
  ```

- Apply the changes.

  ```bash
  kubectl apply -f /root/code/k8s/inference-service.yaml
  ```

- Verify

  The following shows that the Inference Service is Ready.

  ```bash
  kubectl get isvc fraud-detector
  ```
