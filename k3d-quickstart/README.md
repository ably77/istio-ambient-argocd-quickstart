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
For this exercise, we’re going to do all the work on your local workstation. All you’ll need to get started is a Docker-compatible environment such as Docker Desktop, plus the CLI utilities kubectl, k3d, curl, and jq. Make sure these are all available to you before jumping into the next section. I’m building this on MacOS but other platforms should be perfectly fine as well.

### Install k3d

To install k3d simply run the following
```bash
k3d cluster create
```

Verify that the cluster has been created
```bash
kubectl config get-contexts
```

The output should look similar to below
```bash
% kubectl config get-contexts
CURRENT   NAME              CLUSTER           AUTHINFO                NAMESPACE
*         k3d-k3s-default   k3d-k3s-default   admin@k3d-k3s-default   
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
  ignoreDifferences:     
  - group: admissionregistration.k8s.io                                                              
    kind: ValidatingWebhookConfiguration
    jsonPointers:                                                                                    
    - /webhooks/0/clientConfig/caBundle
    - /webhooks/0/failurePolicy
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
  ignoreDifferences:     
  - group: admissionregistration.k8s.io                                                              
    kind: ValidatingWebhookConfiguration
    jsonPointers:                                                                                    
    - /webhooks/0/clientConfig/caBundle
    - /webhooks/0/failurePolicy
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

You can check to see that Istio Ambient architecture components have been deployed
```bash
kubectl get pods -n istio-system && \
kubectl get pods -n kube-system | grep istio-cni
```

Output should look similar to below:
```bash
NAME                      READY   STATUS    RESTARTS   AGE
istiod-759bc56fb5-kn4bp   1/1     Running   0          81s
ztunnel-w879j             1/1     Running   0          51s
istio-cni-node-gzpvd      1/1     Running   0          2m50s
```

# Configure an App

First the client app
```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: client
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/ably77/istio-ambient-argocd-quickstart/
    path: workloads/client/ambient
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m0s
EOF
```

Next we will deploy httpbin
```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: httpbin
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/ably77/istio-ambient-argocd-quickstart/
    path: workloads/httpbin/ambient
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m0s
EOF
```

You can check to see that the applications have been deployed
```bash
kubectl get pods -n client && \
kubectl get pods -n httpbin
```

## exec into sleep client and curl httpbin /get endpoint to verify mTLS
```bash
kubectl exec -it deploy/sleep -n client -c sleep sh

curl httpbin.httpbin.svc.cluster.local:8000/get
```

## Follow ztunnel logs to see mTLS traffic
```bash
kubectl logs -n istio-system ds/ztunnel -f
```

You should see log output similar to below:
```
2024-05-07T04:11:01.169923Z     info    access  connection complete     src.addr=10.244.0.14:42774 src.workload="sleep-9454cc476-hxhx5" src.namespace="client" src.identity="spiffe://cluster.local/ns/client/sa/sleep" dst.addr=10.244.0.15:15008 dst.hbone_addr="10.244.0.15:80" dst.service="httpbin.httpbin.svc.cluster.local" dst.workload="httpbin-698cc5f69-w2rzc" dst.namespace="httpbin" dst.identity="spiffe://cluster.local/ns/httpbin/sa/httpbin" direction="outbound" bytes_sent=104 bytes_recv=467 duration="35ms"
```

## Cleanup

To remove all the Argo CD Applications we configured for this lab
```bash
kubectl delete applications -n argocd httpbin
kubectl delete applications -n argocd client
kubectl delete applications -n argocd istio-ztunnel
kubectl delete applications -n argocd istiod
kubectl delete applications -n argocd istio-cni
kubectl delete applications -n argocd istio-base
```

If you’d like to cleanup the work you’ve done, simply delete the k3d cluster where you’ve been working.
```bash
k3d cluster delete
```

## Learn More
In this blog post, we explored how you can get started with Istio Ambient and Argo CD on your own workstation. We walked step-by-step through the process of standing up a k3d cluster, configuring the new Istio Ambient architecture, installing a couple applications, and then validating zero trust for service-to-service communication without injecting sidecars! All of the code used in this guide is available on github.

A Gloo Mesh Core subscription offers even more value to users who require:

########### change this ###########

Istio Ambient Support;
Istio lifecycle management tooling;
An Insights dashboard;
OTEL Integration

For more information, check out the following resources.
Explore the documentation for Istio Ambient
Request a live demo or trial for Gloo Mesh Core
See video content on the solo.io YouTube channel.
Questions? Join the Solo.io and Istio Slack communities!