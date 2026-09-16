### Task

The xFusionCorp Industries ML platform team has deployed a Kubernetes Deployment of the fraud-detector model server. This deployment uses an `nginx:alpine` container listening on port `80`. Please note that this task focuses on Kubernetes Service routing, rather than the model itself. Currently, the Deployment is operating with two replicas, and executing `kubectl get deploy fraud-detector` confirms that the status shows `READY 2/2`. However, any requests made to the Service at `fraud-detector-svc:8080` result in a timeout. Your task is to diagnose the issue preventing the Service from routing traffic to its backing pods and to correct the manifest accordingly.

1. A kind cluster named `mlops` is pre-provisioned and `kubectl` is configured to reach it. The Deployment and Service manifests live at `/root/code/k8s/deployment.yaml` and `/root/code/k8s/service.yaml`, and both have already been applied — the `fraud-detector` Deployment reports `READY 2/2`, but requests to `fraud-detector-svc:8080` time out. Inspect the Service and its endpoints to see why traffic never reaches the pods:

   ```bash
   kubectl describe svc fraud-detector-svc
   ```

2. The end state must include:
   - Deployment `fraud-detector` is Available.
   - Service `fraud-detector-svc` exists; clients still dial it on `8080`.
   - `Endpoints/fraud-detector-svc` routes to the port the container is actually listening on (`80`).
   - An in-cluster HTTP `GET http://fraud-detector-svc:8080/` returns the nginx default page.

In a Kubernetes Service, `port` is what clients dial and `targetPort` is what kube-proxy forwards that traffic to on the backing pods. They can legitimately differ (e.g. external 8080 → internal 80), but `targetPort` must match a real listening port on the container.

### Solution

- Update the `/root/code/k8s/service.yml`

  ```yml
  apiVersion: v1
  kind: Service
  metadata:
    name: fraud-detector-svc
  spec:
    type: NodePort
    selector:
      app: fraud-detector
    ports:
      - port: 8080
        # targetPort bug -- nginx:alpine listens on 80, not 8080.
        targetPort: 80
        nodePort: 30092
  ```

- Apply the change

  ```bash
  kubectl apply -f /root/code/k8s/service.yaml
  ```

- Verify

  ```bash
  kubectl run curl-test --rm -it --restart=Never --image=curlimages/curl -- curl -v http://fraud-detector-svc:8080/
  ```

  This should output the nginx default page.
