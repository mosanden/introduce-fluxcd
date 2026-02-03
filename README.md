# FluxCD Live Demo

## From Argo CD to Flux

**The Philosophy:** While Argo CD is a developer-centric hub with a rich UI, **FluxCD** is a set of specialized, modular controllers.

**Key Differences:**
* **Architecture:** Flux is "GitOps by Design" (no central API server/SSO needed unless using a UI add-on).
* **CRD Centric:** Every action in Flux is a Kubernetes Custom Resource.
* **The Pull Model:** Flux lives inside the cluster and pulls manifests without external access requirements.

![Architecture - https://fluxcd.io/flux/components/](https://fluxcd.io/img/diagrams/gitops-toolkit.png)

## Preparation
Create an empty git repository:
```bash
cd ~/projects/flux-demo
git remote show origin
git reset --hard 384b3c1f9861ab67e3f1a6c93a2913e154d15461
git push --force
```

## Phase 1: The Bootstrap
Flux initializes itself and manages its own lifecycle. Installation via Helm is also possible.
```bash
# Start a cluster
kind create cluster --name flux-demo

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-demo \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal \
  --private=true

kubectl get namespace
kubectl get pods -n flux-system

# Show all Flux resources
flux get all
# Show logs of the Flux controllers
flux logs
```

The following files will be created in `./clusters/my-cluster/flux-system/`:

**`gotk-components.yaml`**: flux-system Namespace, Deployments of source-controller and kustomize-controller, CRDs, ...

**`gotk-sync.yaml`**: flux-system GitRepository and Kustomization

**`kustomization.yaml`**: K8s-Kustomization referencing `gotk-sync.yaml` and `gotk-components.yaml`

## Important CRDs

### GitRepository 
Reference to a Git repository.
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: ssh://git@github.com/mosanden/flux-demo
```

### Kustomization
Reference to a path in a Git repo containing a K8s-Kustomization. This will be applied to the cluster.
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/my-cluster
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

### HelmRepository and HelmRelease
A HelmRepository defines the source of a Helm Chart (similar to `helm repo add`). A HelmRelease references a Chart in a HelmRepository and installs the Chart in the cluster. (`helm upgrade --install`). Values can be specified in the resource.

The "Real Helm" Experience: Unlike other tools that merely render Helm templates into plain manifests, Flux uses a dedicated Helm Controller. This means it performs actual Helm operations, maintaining the release history and allowing you to use standard Helm tooling alongside GitOps.
```bash
# Verify that it's a real Helm release with full lifecycle tracking
helm ls -A
helm history podinfo -n default
```

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 15m
  url: https://stefanprodan.github.io/podinfo
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 15m
  timeout: 5m
  chart:
    spec:
      chart: podinfo
      version: '6.5.*'
      sourceRef:
        kind: HelmRepository
        name: podinfo
      interval: 5m
  releaseName: podinfo
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  test:
    enable: true
  driftDetection:
    mode: enabled
    ignore:
    - paths: ["/spec/replicas"]
      target:
        kind: Deployment
  values:
    replicaCount: 2
```
https://fluxcd.io/flux/components/helm/helmreleases/#example

### Bucket
Like a GitRepository but references an S3 bucket. Can also be referenced in a Kustomization.
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: Bucket
metadata:
  name: minio-bucket
  namespace: default
spec:
  interval: 5m0s
  endpoint: minio.example.com
  insecure: true
  secretRef:
    name: minio-bucket-secret
  bucketName: example
```
https://fluxcd.io/flux/components/source/buckets/#example

## Phase 2: Adding resources
`./gitops-repo-structure` shows a common way of structuring a GitOps repo with Flux and contains a demo application.
```bash
cp -r ./gitops-repo-structure/* ~/projects/flux-demo
```
Commit the changes. Leave out `capacitor.yaml` and `weave-gitops-dashboard.yaml` for now.

```bash
flux reconcile source git flux-system

flux logs
# 2026-01-27T12:51:41.486Z info GitRepository/flux-system.flux-system - stored artifact for commit 'Add apps, infrastructure, podinfo'

# get all resources from all namespaces
flux get all -A

# get specific resources
flux get kustomization
flux get source helm -n default
flux get helmrelease -n default
```

## Phase 3: Upgrading Flux
To upgrade Flux just update the resources in the Git repo by running:
```bash
flux install --export > ./clusters/my-cluster/flux-system/gotk-components.yaml
```
Alternatively, updates can be installed using Helm or by **rerunning the bootstrap command**. Make sure you have the corresponding version of the Flux CLI installed.

## Phase 4: Installing a UI
There is no official UI for Flux as there is for Argo CD. But there are a couple of third-party options listed [here](https://fluxcd.io/flux/#flux-uis).

### Capacitor
Add and commit `capacitor.yaml`. Then start a port forward.
```bash
kubectl -n flux-system port-forward svc/capacitor 9000:9000
```

### Weave GitOps UI
Add and commit `weave-gitops-dashboard.yaml`. Then start a port forward.
```bash
kubectl -n flux-system port-forward svc/ww-gitops-weave-gitops 9001:9001
```

## Links
- [Flux Configuration](https://fluxcd.io/flux/installation/configuration/)
- [Automate image updates to Git](https://fluxcd.io/flux/guides/image-update/)
