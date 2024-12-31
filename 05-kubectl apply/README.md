# kubectl apply


```shell
kubectl get deployments
kubectl edit deployment productpage 
```

Notice `spec`, `selector`, and `labels`

Change to 2 replicas

```shell
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

Much like `kubectl edit`

```shell
kubectl get deployment reviews -o yaml
kubectl get deployment reviews -o yaml > reviews.yaml
```

Change to 2 replicas and save.

```shell 
kubectl apply -f reviews.yaml
```

`kubectl apply` is the main way we'll interact with K8s resources and workloads.

Repeat:

```shell 
kubectl apply -f reviews.yaml
```

error

Edit to pare down to

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: reviews
  name: reviews
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: reviews
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: reviews
    spec:
      containers:
      - image: docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
        imagePullPolicy: IfNotPresent
        name: examples-bookinfo-reviews-v1
      restartPolicy: Always
      securityContext: {}
```

Repeat:

```shell
kubectl apply -f reviews.yaml
```

works cleaner

Repeat again:

```shell
kubectl apply -f reviews.yaml
```

idempotent

```shell
kubectl explain deployments
kubectl explain deployments --recursive
kubectl explain deployments.spec.template.spec.containers
```
