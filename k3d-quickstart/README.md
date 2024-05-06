# From Zero to Istio Ambient + Argo CD on K3D in 15 minutes!

![15-minutes-header](../.images/15-minutes-header.png)
*…your mileage may vary

## Introduction
How long does it take to configure your first local development environment, sidecarless Service Mesh, and GitOps workflow?

How about 15 minutes? Give us that much time and we’ll give you an ephemeral testbed to explore the benefits of the new architecture mode in Istio, deploy a few applications, and validate zero-trust with zero-effort! We’ll host all of this on a local K3D cluster to keep the setup standalone and as simple as possible.

Would you prefer to perform this exercise on a public cloud rather than a local KinD cluster? Then check out these alternative versions of this post:

From Zero to Istio Ambient + Argo CD on GCP in 15 minutes!
From Zero to Istio Ambient + Argo CD on EKS in 15 minutes!
From Zero to Istio Ambient + Argo CD on AKS in 15 minutes!
If you have questions, please reach out on the Solo Slack channel.

Ready? Set? Go!

## Prerequisites
For this exercise, we’re going to do all the work on your local workstation. All you’ll need to get started is a Docker-compatible environment such as Docker Desktop, plus the CLI utilities kubectl, kind, curl, and jq. Make sure these are all available to you before jumping into the next section. I’m building this on MacOS but other platforms should be perfectly fine as well.

# Configure Docker network
```bash
network=k3d-cluster-network
docker network create "$network" > /dev/null 2>&1 || true
```

### Install k3d

To install k3d simply run the following
```bash
k3d cluster create --wait --config - <<EOF
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: ambient-demo
image: rancher/k3s:v1.28.7-k3s1
network: k3d-cluster-network
ports:
  - port: 8090:8090
    nodeFilters:
      - loadbalancer
  - port: 80:80
    nodeFilters:
      - loadbalancer
  - port: 443:443
    nodeFilters:
      - loadbalancer
options:
  k3d:
    wait: true
    timeout: "60s"
    disableLoadbalancer: false
  k3s:
    extraArgs:
      - arg: --disable=traefik
        nodeFilters:
          - server:*
    nodeLabels:
      - label: topology.kubernetes.io/region=us-west
        nodeFilters:
          - server:*
      - label: topology.kubernetes.io/zone=us-west-1a
        nodeFilters:
          - server:*
  kubeconfig:
    updateDefaultKubeconfig: true
    switchCurrentContext: true
EOF
```

Verify that the cluster has been created
```bash
kubectl config get-contexts
```

The output should look similar to below
```bash
% kubectl config get-contexts
CURRENT   NAME               CLUSTER            AUTHINFO                 NAMESPACE
*         k3d-ambient-demo   k3d-ambient-demo   admin@k3d-ambient-demo  
```

### Installing Argo CD	
Let's start by deploying Argo CD

Deploy Argo CD 2.9.5 using the [non-HA YAML manifests](https://github.com/argoproj/argo-cd/releases)
<!-- Using https://github.com/solo-io/gitops-library/tree/main/argocd/deploy/default -->
```bash
kubectl create namespace argocd

until kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.9.5/manifests/install.yaml > /dev/null 2>&1; do sleep 2; done
```

Check deployment status:
```bash
kubectl -n argocd rollout status deploy/argocd-applicationset-controller
kubectl -n argocd rollout status deploy/argocd-dex-server
kubectl -n argocd rollout status deploy/argocd-notifications-controller
kubectl -n argocd rollout status deploy/argocd-redis
kubectl -n argocd rollout status deploy/argocd-repo-server
kubectl -n argocd rollout status deploy/argocd-server
```

Check to see Argo CD status.
```bash
kubectl get pods -n argocd
```

Output should look similar to below
```bash
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          31s
argocd-applicationset-controller-765944f45d-569kn   1/1     Running   0          33s
argocd-dex-server-7977459848-swnm6                  1/1     Running   0          33s
argocd-notifications-controller-6587c9d9-6t4hg      1/1     Running   0          32s
argocd-redis-b5d6bf5f5-52mk5                        1/1     Running   0          32s
argocd-repo-server-7bfc968f69-hrqt6                 1/1     Running   0          32s
argocd-server-77f84bfbb8-lgdlr                      2/2     Running   0          32s
```

We can also change the password to: `admin / solo.io`:
```bash
# bcrypt(password)=$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy
# password: solo.io
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

Note: if you want to change the password to something else, [follow these instructions in the Argo CD Docs](https://argo-cd.readthedocs.io/en/stable/faq/#i-forgot-the-admin-password-how-do-i-reset-it)

### Access Argo CD with port-forwarding
At this point, we should be able to access our Argo CD server using port-forward at http://localhost:9999
```
kubectl port-forward svc/argocd-server -n argocd 9999:443
```

## Installing Istio Ambient
Set the Istio version in an environment variable
```bash
export ISTIO_VERSION=1.22.0-beta.1
```

Reminder if you want a specific version of Istio or to use the officially supported images provided by Solo.io, get the Hub value from the [Solo support page for Istio Solo images](https://support.solo.io/hc/en-us/articles/4414409064596). The value is present within the `Solo.io Istio Versioning Repo key` section

Otherwise, we can use the upstream Istio community image as defined.

Configure the Kubernetes Gateway API CRDs on the cluster, we will need these to deploy components like the Waypoint proxy
```bash
echo "installing Kubernetes Gateway CRDs"
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0" | kubectl apply -f -; }
```

Deploy the `istio-base` helm chart using Argo CD

```bash
kubectl apply -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-base
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  project: default
  source:
    chart: base
    repoURL: https://istio-release.storage.googleapis.com/charts
    targetRevision: "${ISTIO_VERSION}"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

Deploy the `istio-cni` helm chart using Argo CD

```bash
kubectl apply -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-cni
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  project: default
  source:
    chart: cni
    repoURL: https://istio-release.storage.googleapis.com/charts
    targetRevision: ${ISTIO_VERSION}
    helm:
      values: |
        profile: ambient
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

Deploy the `ztunnel` helm chart using Argo CD

```bash
kubectl apply -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-ztunnel
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  project: default
  source:
    chart: ztunnel
    repoURL: https://istio-release.storage.googleapis.com/charts
    targetRevision: ${ISTIO_VERSION}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

Deploy the `istiod` helm chart using Argo CD

```bash
kubectl apply -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istiod
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  project: default
  source:
    chart: istiod
    repoURL: https://istio-release.storage.googleapis.com/charts
    targetRevision: ${ISTIO_VERSION}
    helm:
      values: |
        profile: ambient
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```


You can check to see that Istio Ambient architecture components have been deployed
```bash
kubectl get pods -n istio-system && \
kubectl get pods -n kube-system
```

Output should look similar to below:
```bash
NAME                             READY   STATUS    RESTARTS   AGE
istiod-1-20-6-86499c5945-bbsfl   1/1     Running   0          38m
NAME                                           READY   STATUS    RESTARTS   AGE
istio-ingressgateway-1-20-6-6575484979-5fbn7   1/1     Running   0          36m
```


