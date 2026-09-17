### Task

The xFusionCorp Industries ML platform team has autoscaled their `fraud-server` Deployment, which serves as a stand-in for the fraud-detector model using `nginx:alpine`. This task focuses on the configuration of the HorizontalPodAutoscaler (HPA), rather than the model itself. Currently, executing `kubectl get hpa` results in `TARGETS <unknown>/70%`, indicating that the HPA is unable to find a metric to measure, and as a consequence, it is not performing any scaling operations. Please investigate the issue and rectify the HPA manifest located at `/root/code/k8s/hpa.yaml`.

1. A kind cluster is pre-provisioned and `kubectl` is configured to reach it. `metrics-server` is installed and patched for kind's kubelet, the `fraud-server` Deployment is running with CPU requests and limits, and the HPA manifest at `/root/code/k8s/hpa.yaml` has already been applied — its `TARGETS` column currently reads `<unknown>/70%`. Inspect the HPA to see why it can't read a metric:

   ```bash
   kubectl describe hpa fraud-server-hpa
   ```

2. The end state must include:
   - The `fraud-server` Deployment is Available with 2 or more replicas.
   - HPA `fraud-server-hpa` exists, and its `scaleTargetRef` points at a Deployment that exists in the cluster.
   - `HPA.status.currentMetrics[].resource.current.averageUtilization` (or `averageValue`) is populated — no longer `<unknown>` (tests poll up to 180 s).

An HPA is a reference plus a metric target — nothing works if the reference points at a resource that no longer exists. Silent break modes like this are why CI gates on `kubectl apply --dry-run=server` (which validates the scaleTargetRef against live resources) are a useful safety net in pipelines that rename workloads.

### Solution

- Inspect the HPA to identify the problem

  ```bash
  kubectl describe hpa fraud-server-hpa
  ```

  It will indicate the problem is with the name of the `scaleTargetRef`.

- Fix the error and apply the changes.

  Change the name to `fraud-server`.

  ```yml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: fraud-server-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      # scaleTargetRef bug -- real name is `fraud-server`.
      name: fraud-server
    minReplicas: 2
    maxReplicas: 10
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70
  ```

  ```bash
  kubectl apply -f /root/code/k8s/hpa.yaml
  ```

- Verify

  The following list the `fraud-server` Deployment is Available with 2 or more replicas.

  ```bash
  kubectl get deploy
  ```

  The following shows that its targets are showing

  ```bash
  kubectl get hpa
  ```
