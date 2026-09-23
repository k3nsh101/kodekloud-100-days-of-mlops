### Task

The xFusionCorp Industries MLOps team is building the GitOps deployment layer for their `fraud-detector` model server (an `nginx` stand-in — the rollout loop is image-agnostic). The server's Kubernetes manifests live in a Gitea repo, and ArgoCD should reconcile the cluster against that repo. The kind cluster, in-cluster Gitea with the seeded `mlops-deploy` repo, and ArgoCD are all in place, but no Application exists yet. Your task is to wire and drive the loop: complete `application.yaml` so ArgoCD tracks the `mlops-deploy` manifests and apply it, sync it to deploy the server, then roll a new image version by bumping the tag in the Gitea repo and syncing again.

1. The **Gitea UI** (port `3000`) and the **ArgoCD UI** (port `5000`) buttons at the top of the lab open the relevant UIs. Both accept ArgoCD `admin` / `adminadmin`, Gitea `gitops-admin` / `adminadmin`. Pre-staged state:
   - Gitea repo `gitops-admin/mlops-deploy` contains `manifests/deployment.yaml` (image `nginx:1.25-alpine`) and `manifests/service.yaml` (NodePort `30080`, exposed on host `:8085`).
   - ArgoCD is installed with the `mlops-deploy` repository already registered, but no **Application exists yet** — you create it.
   - `/root/code/application.yaml` is a scaffold with three `TODOs`. Fill them so the Application tracks the repo:
   - repo URL: `http://gitea-http.gitea.svc.cluster.local:3000/gitops-admin/mlops-deploy.git`
   - manifests path: `manifests`
   - destination namespace: `default`

2. The target version is `nginx:1.27-alpine`. The end state must include:
   - An ArgoCD Application `fraud-detector` exists and is `Synced` + `Healthy` (tests poll up to 240 s).
   - `manifests/deployment.yaml` in the Gitea repo references `nginx:1.27-alpine`.
   - The `fraud-detector` Deployment in default runs image `nginx:1.27-alpine`.
   - `http://localhost:8085/` returns HTTP `200` from the running pod.

The source of truth is the Gitea repo — the rollout runs through the Gitea web editor and the ArgoCD UI. Reference manifests also live under `/root/code/manifests/` for transparency, but edits to those local files are not tracked and will not roll out.

### Solution

- Update the `/root/code/application.yml`.

  ```yml
  # ArgoCD Application for the fraud-detector model server.
  #
  # Fill the three TODOs (see the task for the repo URL, manifests path,
  # and target namespace), then apply it so ArgoCD reconciles the cluster
  # against the mlops-deploy repo:
  #   kubectl apply -n argocd -f /root/code/application.yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: fraud-detector
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: "http://gitea-http.gitea.svc.cluster.local:3000/gitops-admin/mlops-deploy.git" # TODO: the mlops-deploy repository's git URL
      targetRevision: HEAD
      path: "manifests" # TODO: the folder in the repo holding the manifests
    destination:
      server: https://kubernetes.default.svc
      namespace: "default" # TODO: the namespace the manifests deploy into
  ```

- Apply the manifest.

  ```bash
  kubectl apply -n argocd -f application.yaml
  ```

- Log into **ArgoCD UI** and sync the `fraud-detector` application.

- Log into **Gitea UI** and edit the `manifests/deployment.yml` in `mlops-deploy` repo as follows.

  Update the nginx version and commit changes.

  ```yml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: fraud-detector
    namespace: default
    labels:
      app: fraud-detector
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: fraud-detector
    template:
      metadata:
        labels:
          app: fraud-detector
      spec:
        containers:
          - name: fraud-detector
            image: nginx:1.27-alpine
            ports:
              - containerPort: 80
  ```

- Refresh and Resync the `fraud-detector` application from **ArgoCD UI**.

- Verify `http://localhost:8085/` returns HTTP 200 from the running pod.

  ```bash
  curl -I http://localhost:8085
  ```
