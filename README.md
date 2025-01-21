# argocd-applications

This repository contains examples with ArgoCD.

## Installing ArgoCD

Install Arco CD in a k8s cluster locally using `kind`

```bash
    brew install kind
    kind create cluster
```

### Deploying Argo CD Using YAML Manifests

```bash
    kubectl create namespace argocd

    # install specific version
    # kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.3/manifests/install.yaml

    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

    kubectl port-forward svc/argocd-server -n argocd 8080:443
    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d

```

To delete and recreate the kind cluster, using the following commands:

```bash
  kind delete cluster
  Kind create cluster
```

Alternatively, instead of needing to recreate the entire kind cluster, the YAML based manifest installation of Argo CD can be uninstalled by removing the resources from the same manifest and then deleting the argocd namespace:

```bash
  # delete specific version
  # kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.3/manifests/install.yaml

  kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  kubectl delete namespace argocd
```

One of the most common methods for exposing services and gaining access to resources within a Kubernetes cluster is to leverage an Ingress resource. To forward local ports to the kind node – a capability to enable ingress into the Kubernetes cluster and in this case, the NGINX ingress controller.

```bash
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

```

**Note**: Alternatively, the configuration definition can be placed into a file and referenced using the same `--config` parameter when creating the cluster.

Once the cluster has started, deploy the NGINX ingress controller:

```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait until the ingress controller is ready.

```bash
  kubectl wait --namespace ingress-nginx \
    --for=condition=ready pod \
    --selector=app.kubernetes.io/component=controller \
    --timeout=90s
```

Query the pods and services in the ingress-nginx namespace to view the resources that were just deployed

```bash
  kubectl get pods -n ingress-nginx
```

### Deploying Argo CD Using Helm

```bash
# First, add the Argo CD Helm repository:
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade -i argo-cd argo/argo-cd -n argocd --create-namespace

# To view the full set of tunable parameters provided by the Argo CD Helm chart
helm show values argo/argo-cd

# specifying variables
helm upgrade --install --wait --timeout 15m --atomic --namespace argocd --create-namespace \
  --repo https://argoproj.github.io/argo-helm argocd argo-cd --values - <<EOF
dex:
  enabled: false
redis:
  enabled: true
redis-ha:
  enabled: false
repoServer:
  serviceAccount:
    create: true
server:
  config:
    resource.compareoptions: |
      ignoreAggregatedRoles: true
      ignoreResourceStatusField: all
    url: http://localhost/argocd
    application.instanceLabelKey: argocd.argoproj.io/instance
  extraArgs:
    - --insecure
    - --rootpath
    - /argocd
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: ca-issuer
    enabled: true
    paths:
      - /argocd
EOF

```

Since the kind cluster was created binding port 80 and 443 of the local machine to the kind node, invoking the curl command against port 80 should verify communication to the ingress controller

```bash
  curl http://127.0.0.1
  # Not found respone is expected at this point
```

Modify the contents of the `/etc/hosts` file on the local machine

```text
  127.0.0.1 argocd.jsolana.local
```

Create a file called `values-argocd-ingress.yaml` with the following content:

```yml
---
server:
  ingress:
    enabled: true
    hostname: argocd.jsolana.local
    ingressClassName: nginx
  extraArgs:
  - --insecure
```

Deploy the Helm chart using the values file created previously.

```bash
  helm upgrade -i argo-cd argo/argo-cd --namespace argocd --create-namespace -f values-argocd-ingress.yaml
```

A new Ingress resource was created which can be verified by executing the following command:

```bash
  kubectl get ingress -n argocd
```

Given that all of the pieces are in place in order to access Argo CD using an Ingress resource, open a web browser and navigate to `http://argocd.jsolana.local/` which should display the Argo CD User Interface login page or login using argocd cli `argocd login --insecure --grpc-web argocd.jsolana.local`.

The OpenAPI specification provided by Argo CD can be accessed in `https://argocd.jsolana.local/swagger-ui`.

## Application

To create this Argo CD Application within the Kubernetes cluster, you can apply it using the following kubectl command.

```bash
  kubectl apply -f app.yaml
````

`apps` directory contains several examples of applications.

In-line examples:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: busybox-echo
  namespace: argocd
  finalizers:
    # To remove the resources that are managed by an Application
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: busybox-echo
    server: https://kubernetes.default.svc
  project: default
  source:
    path: manifests/busybox
    repoURL: https://github.com/jsolana/argocd-applications
    targetRevision: HEAD
  syncPolicy:
    automated:
      allowEmpty: true
      prune: true
    retry:
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 1m
      limit: 2
    syncOptions:
    - CreateNamespace=true
```

## ApplicationSet

A git generator's applicationset:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: echo-appset
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/jsolana/argocd-applications.git
      revision: HEAD
      directories:
      - path: manifests/busybox/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: "default"
      source:
        repoURL: https://github.com/jsolana/argocd-applications.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          allowEmpty: true
        syncOptions:
        - ApplyOutOfSyncOnly=true
        - Validate=true
        - CreateNamespace=true
        - PrunePropagationPolicy=foreground
        - ServerSideApply=true
        - FailOnSharedResource=true
        retry:
          limit: 1
          backoff:
            duration: 2m
            factor: 2
            maxDuration: 3m
```
