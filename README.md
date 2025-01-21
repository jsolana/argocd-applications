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

## Dry Run

Firs of all, lets gonna create `busybox-echo` app:

```bash
kubectl apply -f apps/busybox-app-yml
```

Second, lets gonna create all this changes in a branch called `dry-run`.

To test dry run first check updating an already existent resource (in this case, the deployment):

```yaml
# we are gona use manifests/busyboz/deployment.yml cause previously we installed busybox-app
# change using latest in a branch called dryrun
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-dev-echo
  labels:
    tier: "3"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-dev-echo
  template:
    metadata:
      labels:
        app: busybox-dev-echo
        tier: "3"
    spec:
      containers:
      - name: busybox
        image: busybox:latest
        command: ["/bin/sh", "-c"]
        args: ["while true; do echo Hello Dev; sleep 5; done"]
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"

```

Execute  argocd cli:

```bash
argocd app sync busybox-echo --revision dry-run --dry-run --timeout 60 --retry-limit 1
```

Here is the log of argocd application controller:

```log
time="2025-01-21T12:17:17Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:17:17Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:17:17Z" level=info msg="Start Update application operation state"
time="2025-01-21T12:17:17Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:17:17Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:17:17Z" level=info msg="Completed Update application operation state"
time="2025-01-21T12:17:17Z" level=info msg="updated 'argocd/busybox-echo' operation (phase: Running)" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:17:17Z" level=info msg="Initialized new operation: {&SyncOperation{Revision:e719c0e7684378b186be3b6f762b604deeb7d0a4,Prune:false,DryRun:true,SyncStrategy:&SyncStrategy{Apply:nil,Hook:&SyncStrategyHook{SyncStrategyApply:SyncStrategyApply{Force:false,},},},Resources:[]SyncOperationResource{},Source:nil,Manifests:[],SyncOptions:[CreateNamespace=true],Sources:[]ApplicationSource{},Revisions:[],} {admin false} [] {1 &Backoff{Duration:5s,Factor:*2,MaxDuration:3m0s,}}}" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:17:17Z" level=info msg="Comparing app state (cluster: https://kubernetes.default.svc, namespace: busybox-echo)" application=argocd/busybox-echo
time="2025-01-21T12:17:17Z" level=debug msg="Generating Manifest for source {https://github.com/jsolana/argocd-applications manifests/busybox HEAD nil nil nil nil  } revision e719c0e7684378b186be3b6f762b604deeb7d0a4"
time="2025-01-21T12:17:17Z" level=info msg="GetRepoObjs stats" application=argocd/busybox-echo build_options_ms=0 helm_ms=0 plugins_ms=0 repo_ms=0 time_ms=13 unmarshal_ms=11 version_ms=0
time="2025-01-21T12:17:17Z" level=debug msg="Retrieved live manifests" application=argocd/busybox-echo
time="2025-01-21T12:17:17Z" level=debug msg=revisionChanged application=argocd/busybox-echo useDiffCache=false
time="2025-01-21T12:17:17Z" level=info msg="Applying resource Deployment/busybox-dev-echo in cluster: https://10.96.0.1:443, namespace: busybox-echo" dry-run=server manager=argocd-controller serverSideApply=true serverSideDiff=true
time="2025-01-21T12:17:17Z" level=info msg="{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"labels\":{\"app.kubernetes.io/instance\":\"busybox-echo\",\"tier\":\"3\"},\"name\":\"busybox-dev-echo\",\"namespace\":\"busybox-echo\"},\"spec\":{\"replicas\":1,\"selector\":{\"matchLabels\":{\"app\":\"busybox-dev-echo\"}},\"template\":{\"metadata\":{\"labels\":{\"app\":\"busybox-dev-echo\",\"tier\":\"3\"}},\"spec\":{\"containers\":[{\"args\":[\"while true; do echo Hello Dev; sleep 5; done\"],\"command\":[\"/bin/sh\",\"-c\"],\"image\":\"busybox:latest\",\"name\":\"busybox\",\"resources\":{\"limits\":{\"cpu\":\"200m\",\"memory\":\"128Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"64Mi\"}}}]}}}}"
time="2025-01-21T12:17:17Z" level=info msg="Start Update application operation state"
time="2025-01-21T12:17:17Z" level=info msg="Completed Update application operation state"
time="2025-01-21T12:17:17Z" level=info msg="updated 'argocd/busybox-echo' operation (phase: Running)" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:17:17Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:17:17Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:17:17Z" level=info msg="Skipping retrying in-progress operation. Attempting again at: 2025-01-21T12:17:27Z" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:17:17Z" level=info msg="Skipping retrying in-progress operation. Attempting again at: 2025-01-21T12:17:27Z" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
```

This ends with an error:

```console
Message:            ComparisonError: Failed to compare desired state to live state: failed to calculate diff: error calculating server side diff: serverSideDiff error: error running server side apply in dryrun mode for resource Deployment/busybox-dev-echo: admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Deployment/busybox-echo/busybox-dev-echo was blocked due to the following policies 

disallow-latest-tag:
  autogen-validate-image-tag: 'validation failure: validation error: Using a mutable
    image tag e.g. ''latest'' is not allowed. rule autogen-validate-image-tag failed
    at path /image/' (retried 1 times).
```

Lets undo and test with a new resource (non existent yet):

```log
time="2025-01-21T12:22:11Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:22:11Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:22:11Z" level=info msg="Start Update application operation state"
time="2025-01-21T12:22:11Z" level=info msg="Completed Update application operation state"
time="2025-01-21T12:22:11Z" level=info msg="updated 'argocd/busybox-echo' operation (phase: Running)" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:22:11Z" level=info msg="Initialized new operation: {&SyncOperation{Revision:1a5ed6793fad7f757af578923b901cc5e930ad07,Prune:false,DryRun:true,SyncStrategy:&SyncStrategy{Apply:nil,Hook:&SyncStrategyHook{SyncStrategyApply:SyncStrategyApply{Force:false,},},},Resources:[]SyncOperationResource{},Source:nil,Manifests:[],SyncOptions:[CreateNamespace=true],Sources:[]ApplicationSource{},Revisions:[],} {admin false} [] {1 &Backoff{Duration:5s,Factor:*2,MaxDuration:3m0s,}}}" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:22:11Z" level=info msg="Comparing app state (cluster: https://kubernetes.default.svc, namespace: busybox-echo)" application=argocd/busybox-echo
time="2025-01-21T12:22:11Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:22:11Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:22:11Z" level=debug msg="Generating Manifest for source {https://github.com/jsolana/argocd-applications manifests/busybox HEAD nil nil nil nil  } revision 1a5ed6793fad7f757af578923b901cc5e930ad07"
time="2025-01-21T12:22:12Z" level=info msg="GetRepoObjs stats" application=argocd/busybox-echo build_options_ms=0 helm_ms=2 plugins_ms=0 repo_ms=0 time_ms=836 unmarshal_ms=833 version_ms=0
time="2025-01-21T12:22:12Z" level=debug msg="Retrieved live manifests" application=argocd/busybox-echo
time="2025-01-21T12:22:12Z" level=debug msg=revisionChanged application=argocd/busybox-echo useDiffCache=false
time="2025-01-21T12:22:12Z" level=info msg="Applying resource Deployment/busybox-dev-echo in cluster: https://10.96.0.1:443, namespace: busybox-echo" dry-run=server manager=argocd-controller serverSideApply=true serverSideDiff=true
time="2025-01-21T12:22:12Z" level=info msg="{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"labels\":{\"app.kubernetes.io/instance\":\"busybox-echo\",\"tier\":\"3\"},\"name\":\"busybox-dev-echo\",\"namespace\":\"busybox-echo\"},\"spec\":{\"replicas\":1,\"selector\":{\"matchLabels\":{\"app\":\"busybox-dev-echo\"}},\"template\":{\"metadata\":{\"labels\":{\"app\":\"busybox-dev-echo\",\"tier\":\"3\"}},\"spec\":{\"containers\":[{\"args\":[\"while true; do echo Hello Dev; sleep 5; done\"],\"command\":[\"/bin/sh\",\"-c\"],\"image\":\"busybox\",\"name\":\"busybox\",\"resources\":{\"limits\":{\"cpu\":\"200m\",\"memory\":\"128Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"64Mi\"}}}]}}}}"
time="2025-01-21T12:22:12Z" level=info msg=Syncing application=argocd/busybox-echo skipHooks=false started=false syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg="Tasks from managed resources" application=argocd/busybox-echo resourceTasks="[Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (,,)]" syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg="tasks from hooks" application=argocd/busybox-echo hookTasks="[]" syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg="Namespace already exists" application=argocd/busybox-echo namespace=busybox-echo syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg="Tasks (dry-run)" application=argocd/busybox-echo syncId=00021-dbRTf tasks="[Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (,,)]"
time="2025-01-21T12:22:12Z" level=info msg="Running tasks" application=argocd/busybox-echo dryRun=true numTasks=1 syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg=Applying application=argocd/busybox-echo dryRun=true syncId=00021-dbRTf task="Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (,,)"
time="2025-01-21T12:22:12Z" level=info msg="Applying resource Deployment/busybox-dev-echo in cluster: https://10.96.0.1:443, namespace: busybox-echo" dry-run=client manager=argocd-controller serverSideApply=false serverSideDiff=false
time="2025-01-21T12:22:12Z" level=info msg="{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"labels\":{\"app.kubernetes.io/instance\":\"busybox-echo\",\"tier\":\"3\"},\"name\":\"busybox-dev-echo\",\"namespace\":\"busybox-echo\"},\"spec\":{\"replicas\":1,\"selector\":{\"matchLabels\":{\"app\":\"busybox-dev-echo\"}},\"template\":{\"metadata\":{\"labels\":{\"app\":\"busybox-dev-echo\",\"tier\":\"3\"}},\"spec\":{\"containers\":[{\"args\":[\"while true; do echo Hello Dev; sleep 5; done\"],\"command\":[\"/bin/sh\",\"-c\"],\"image\":\"busybox\",\"name\":\"busybox\",\"resources\":{\"limits\":{\"cpu\":\"200m\",\"memory\":\"128Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"64Mi\"}}}]}}}}"
time="2025-01-21T12:22:12Z" level=info msg="Adding resource result, status: 'Synced', phase: 'Succeeded', message: 'deployment.apps/busybox-dev-echo configured (dry run)'" application=argocd/busybox-echo kind=Deployment name=busybox-dev-echo namespace=busybox-echo phase=Sync syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg="Filtering out non-pending tasks" application=argocd/busybox-echo syncId=00021-dbRTf tasks="[Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (Synced,Succeeded,deployment.apps/busybox-dev-echo configured (dry run))]"
time="2025-01-21T12:22:12Z" level=info msg="Updating operation state. phase: Running -> Succeeded, message: '' -> 'successfully synced (no more tasks)'" application=argocd/busybox-echo syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg="sync/terminate complete" application=argocd/busybox-echo duration=47.079916ms syncId=00021-dbRTf
time="2025-01-21T12:22:12Z" level=info msg="Start Update application operation state"
time="2025-01-21T12:22:12Z" level=info msg="Completed Update application operation state"
time="2025-01-21T12:22:12Z" level=info msg="updated 'argocd/busybox-echo' operation (phase: Succeeded)" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:22:12Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:22:12Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:22:12Z" level=info msg="Sync operation to 1a5ed6793fad7f757af578923b901cc5e930ad07 succeeded" application=busybox-echo dest-namespace=busybox-echo dest-server="https://kubernetes.default.svc" reason=OperationCompleted type=Normal
```

This execution ends ok:

```console
Message:            successfully synced (no more tasks)

GROUP  KIND        NAMESPACE     NAME              STATUS  HEALTH   HOOK  MESSAGE
apps   Deployment  busybox-echo  busybox-dev-echo  Synced  Healthy        deployment.apps/busybox-dev-echo configured (dry run)
```

Lets create a new resource and lets see the execution:

```yaml
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
  namespace: busybox-echo
  labels:
    app: helloworld
spec:
  serviceName: "nginx-service"
  replicas: 2
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

```console
time="2025-01-21T12:25:59Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:25:59Z" level=debug msg="Successfully saved info of 1 clusters"
time="2025-01-21T12:25:59Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:25:59Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:25:59Z" level=info msg="Start Update application operation state"
time="2025-01-21T12:25:59Z" level=info msg="Completed Update application operation state"
time="2025-01-21T12:25:59Z" level=info msg="updated 'argocd/busybox-echo' operation (phase: Running)" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:25:59Z" level=info msg="Initialized new operation: {&SyncOperation{Revision:11f592acc1ac1a6fae3ab8c238a5fceaf1add611,Prune:false,DryRun:true,SyncStrategy:&SyncStrategy{Apply:nil,Hook:&SyncStrategyHook{SyncStrategyApply:SyncStrategyApply{Force:false,},},},Resources:[]SyncOperationResource{},Source:nil,Manifests:[],SyncOptions:[CreateNamespace=true],Sources:[]ApplicationSource{},Revisions:[],} {admin false} [] {1 &Backoff{Duration:5s,Factor:*2,MaxDuration:3m0s,}}}" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:25:59Z" level=info msg="Comparing app state (cluster: https://kubernetes.default.svc, namespace: busybox-echo)" application=argocd/busybox-echo
time="2025-01-21T12:25:59Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:25:59Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
time="2025-01-21T12:25:59Z" level=debug msg="Generating Manifest for source {https://github.com/jsolana/argocd-applications manifests/busybox HEAD nil nil nil nil  } revision 11f592acc1ac1a6fae3ab8c238a5fceaf1add611"
time="2025-01-21T12:25:59Z" level=info msg="GetRepoObjs stats" application=argocd/busybox-echo build_options_ms=0 helm_ms=2 plugins_ms=0 repo_ms=0 time_ms=20 unmarshal_ms=17 version_ms=0
time="2025-01-21T12:25:59Z" level=debug msg="Retrieved live manifests" application=argocd/busybox-echo
time="2025-01-21T12:25:59Z" level=debug msg=revisionChanged application=argocd/busybox-echo useDiffCache=false
time="2025-01-21T12:25:59Z" level=info msg="Applying resource Deployment/busybox-dev-echo in cluster: https://10.96.0.1:443, namespace: busybox-echo" dry-run=server manager=argocd-controller serverSideApply=true serverSideDiff=true
time="2025-01-21T12:25:59Z" level=info msg="{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"labels\":{\"app.kubernetes.io/instance\":\"busybox-echo\",\"tier\":\"3\"},\"name\":\"busybox-dev-echo\",\"namespace\":\"busybox-echo\"},\"spec\":{\"replicas\":1,\"selector\":{\"matchLabels\":{\"app\":\"busybox-dev-echo\"}},\"template\":{\"metadata\":{\"labels\":{\"app\":\"busybox-dev-echo\",\"tier\":\"3\"}},\"spec\":{\"containers\":[{\"args\":[\"while true; do echo Hello Dev; sleep 5; done\"],\"command\":[\"/bin/sh\",\"-c\"],\"image\":\"busybox\",\"name\":\"busybox\",\"resources\":{\"limits\":{\"cpu\":\"200m\",\"memory\":\"128Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"64Mi\"}}}]}}}}"
time="2025-01-21T12:25:59Z" level=info msg=Syncing application=argocd/busybox-echo skipHooks=false started=false syncId=00023-HAyFy
time="2025-01-21T12:25:59Z" level=info msg="Tasks from managed resources" application=argocd/busybox-echo resourceTasks="[Sync/0 resource apps/StatefulSet:busybox-echo/nginx-statefulset nil->obj (,,), Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (,,)]" syncId=00023-HAyFy
time="2025-01-21T12:25:59Z" level=info msg="tasks from hooks" application=argocd/busybox-echo hookTasks="[]" syncId=00023-HAyFy
time="2025-01-21T12:25:59Z" level=info msg="Namespace already exists" application=argocd/busybox-echo namespace=busybox-echo syncId=00023-HAyFy
time="2025-01-21T12:25:59Z" level=info msg="Tasks (dry-run)" application=argocd/busybox-echo syncId=00023-HAyFy tasks="[Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (,,), Sync/0 resource apps/StatefulSet:busybox-echo/nginx-statefulset nil->obj (,,)]"
time="2025-01-21T12:25:59Z" level=info msg="Running tasks" application=argocd/busybox-echo dryRun=true numTasks=2 syncId=00023-HAyFy
time="2025-01-21T12:25:59Z" level=info msg=Applying application=argocd/busybox-echo dryRun=true syncId=00023-HAyFy task="Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (,,)"
time="2025-01-21T12:25:59Z" level=info msg="Applying resource Deployment/busybox-dev-echo in cluster: https://10.96.0.1:443, namespace: busybox-echo" dry-run=client manager=argocd-controller serverSideApply=false serverSideDiff=false
time="2025-01-21T12:25:59Z" level=info msg="{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"labels\":{\"app.kubernetes.io/instance\":\"busybox-echo\",\"tier\":\"3\"},\"name\":\"busybox-dev-echo\",\"namespace\":\"busybox-echo\"},\"spec\":{\"replicas\":1,\"selector\":{\"matchLabels\":{\"app\":\"busybox-dev-echo\"}},\"template\":{\"metadata\":{\"labels\":{\"app\":\"busybox-dev-echo\",\"tier\":\"3\"}},\"spec\":{\"containers\":[{\"args\":[\"while true; do echo Hello Dev; sleep 5; done\"],\"command\":[\"/bin/sh\",\"-c\"],\"image\":\"busybox\",\"name\":\"busybox\",\"resources\":{\"limits\":{\"cpu\":\"200m\",\"memory\":\"128Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"64Mi\"}}}]}}}}"
time="2025-01-21T12:25:59Z" level=info msg="Adding resource result, status: 'Synced', phase: 'Succeeded', message: 'deployment.apps/busybox-dev-echo configured (dry run)'" application=argocd/busybox-echo kind=Deployment name=busybox-dev-echo namespace=busybox-echo phase=Sync syncId=00023-HAyFy
time="2025-01-21T12:25:59Z" level=info msg=Applying application=argocd/busybox-echo dryRun=true syncId=00023-HAyFy task="Sync/0 resource apps/StatefulSet:busybox-echo/nginx-statefulset nil->obj (,,)"
time="2025-01-21T12:25:59Z" level=info msg="Applying resource StatefulSet/nginx-statefulset in cluster: https://10.96.0.1:443, namespace: busybox-echo" dry-run=client manager=argocd-controller serverSideApply=false serverSideDiff=false
time="2025-01-21T12:25:59Z" level=info msg="{\"apiVersion\":\"apps/v1\",\"kind\":\"StatefulSet\",\"metadata\":{\"labels\":{\"app\":\"helloworld\",\"app.kubernetes.io/instance\":\"busybox-echo\"},\"name\":\"nginx-statefulset\",\"namespace\":\"busybox-echo\"},\"spec\":{\"replicas\":2,\"selector\":{\"matchLabels\":{\"app\":\"helloworld\"}},\"serviceName\":\"nginx-service\",\"template\":{\"metadata\":{\"labels\":{\"app\":\"helloworld\"}},\"spec\":{\"containers\":[{\"image\":\"nginx:latest\",\"name\":\"nginx\",\"ports\":[{\"containerPort\":80}]}]}}}}"
time="2025-01-21T12:26:00Z" level=info msg="Adding resource result, status: 'Synced', phase: 'Succeeded', message: 'statefulset.apps/nginx-statefulset created (dry run)'" application=argocd/busybox-echo kind=StatefulSet name=nginx-statefulset namespace=busybox-echo phase=Sync syncId=00023-HAyFy
time="2025-01-21T12:26:00Z" level=info msg="Filtering out non-pending tasks" application=argocd/busybox-echo syncId=00023-HAyFy tasks="[Sync/0 resource apps/Deployment:busybox-echo/busybox-dev-echo obj->obj (Synced,Succeeded,deployment.apps/busybox-dev-echo configured (dry run)), Sync/0 resource apps/StatefulSet:busybox-echo/nginx-statefulset nil->obj (Synced,Succeeded,statefulset.apps/nginx-statefulset created (dry run))]"
time="2025-01-21T12:26:00Z" level=info msg="Updating operation state. phase: Running -> Succeeded, message: '' -> 'successfully synced (no more tasks)'" application=argocd/busybox-echo syncId=00023-HAyFy
time="2025-01-21T12:26:00Z" level=info msg="sync/terminate complete" application=argocd/busybox-echo duration=69.453243ms syncId=00023-HAyFy
time="2025-01-21T12:26:00Z" level=info msg="Start Update application operation state"
time="2025-01-21T12:26:00Z" level=info msg="Completed Update application operation state"
time="2025-01-21T12:26:00Z" level=info msg="updated 'argocd/busybox-echo' operation (phase: Succeeded)" app-namespace=argocd app-qualified-name=argocd/busybox-echo application=busybox-echo project=default
time="2025-01-21T12:26:00Z" level=info msg="Sync operation to 11f592acc1ac1a6fae3ab8c238a5fceaf1add611 succeeded" application=busybox-echo dest-namespace=busybox-echo dest-server="https://kubernetes.default.svc" reason=OperationCompleted type=Normal
time="2025-01-21T12:26:00Z" level=debug msg="Checking if cluster https://kubernetes.default.svc with clusterShard 0 should be processed by shard 0"
time="2025-01-21T12:26:00Z" level=debug msg="Skipping sharding distribution update. No relevant changes"
```

:warning: Expected to fail but it is ok:

```console
Message:            successfully synced (no more tasks)

GROUP  KIND         NAMESPACE     NAME               STATUS     HEALTH   HOOK  MESSAGE
apps   Deployment   busybox-echo  busybox-dev-echo   Synced     Healthy        deployment.apps/busybox-dev-echo configured (dry run)
apps   StatefulSet  busybox-echo  nginx-statefulset  Succeeded  Synced         statefulset.apps/nginx-statefulset created (dry run)
```

Running `kubectl apply -f sts.yml --dry-run=server` trigger the error expected:

```console
Error from server: error when creating "sts.yml": admission webhook "validate.kyverno.svc-fail" denied the request: 

resource StatefulSet/busybox-echo/nginx-statefulset was blocked due to the following policies 

disallow-latest-tag:
  autogen-validate-image-tag: 'validation failure: validation error: Using a mutable
    image tag e.g. ''latest'' is not allowed. rule autogen-validate-image-tag failed
    at path /image/'
validate-labels:
  autogen-require-tier-label: 'validation error: Label `tier` with values 1, 2, 3
    or 4 must be set on all pods. rule autogen-require-tier-label failed at path /spec/template/metadata/labels/tier/'
```

The hypothesis is for new resources, kubectl apply is executed in client mode:

```log
time="2025-01-21T12:25:59Z" level=info msg="Applying resource StatefulSet/nginx-statefulset in cluster: https://10.96.0.1:443, namespace: busybox-echo" dry-run=client manager=argocd-controller serverSideApply=false serverSideDiff=false
```

But not sure why.

Maybe it is related with https://github.com/argoproj/argo-cd/issues/13874  [referenced here](https://github.com/argoproj/gitops-engine/blob/847cfc9f8b200e96a70b591a68b9fb385cf2ce56/pkg/sync/sync_context.go#L970)