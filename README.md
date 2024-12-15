# my-first-kubernetes
An interactive workshop for learning a few Kubernetes basics, for developers that already know the basics of Docker and containers.

- signup and gen envs
- bookinfo via docker
  - discuss k8s capabilities
    - service discovery
    - networking
    - failover/self-healing
    - scaling/load balancing
    - rollouts/rollbacks
    - resource management
- deploy bookinfo in k3s by hand
  - pods
  - kubectl create
  - kubectl expose
  - kubectl delete
  - deployments
- edit bookinfo
  - service
    - ClusterIP
    - NodePort
    - LoadBalancer
    - Ingress
- Nodes
- K3s agents
  - namespaces
  - kubectl apply
  - checkov

- Others
  - kubectl exec
  
## bookinfo via docker

get IP

```shell
#export DETAILS_HOSTNAME=localhost
#export DETAILS_SERVICE_PORT=9082
#export REVIEWS_HOSTNAME=localhost
#export REVIEWS_SERVICE_PORT=9081 should be -e
docker run --rm -d -p 9080:9080 docker.io/istio/examples-bookinfo-productpage-v1:1.20.2
docker run --rm -d -p 9081:9080 docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
docker run --rm -d -p 9082:9080 docker.io/istio/examples-bookinfo-details-v1:1.20.2
```

http://localhost:9080/productpage

## deploy bookinfo in k3s by hand

```shell
kubectl cluster-info
kubectl get pods
kubectl run productpage --image docker.io/istio/examples-bookinfo-productpage-v1:1.20.2

kubectl get pods
# explain pods

kubectl run reviews --image docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
kubectl run details --image docker.io/istio/examples-bookinfo-details-v1:1.20.2

kubectl get pods
kubectl get pods -o wide
```

No networking yet

```shell
kubectl expose pod productpage --port 9080 --target-port 9080
kubectl expose pod reviews --port 9080 --target-port 9080
kubectl expose pod details --port 9080 --target-port 9080
kubectl get services
```

http://10.99.151.9:9080/productpage doesn't work, clusterip
internal only, IP handled only within Kubernetes
NodePort maps a high port 30000–32768 from the cluster itself to a service

```shell
kubectl delete service productpage
kubectl expose pod productpage --port 9080 --target-port 9080 --type NodePort
kubectl get services
```

http://localhost:32746/productpage

```shell
kubectl delete service details productpage reviews
kubectl delete pod details productpage reviews
```

pods vs deployments

instead of `kubectl run` use `kubectl create deployment`

```shell
kubectl create deployment productpage --image docker.io/istio/examples-bookinfo-productpage-v1:1.20.2
kubectl create deployment reviews --image docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
kubectl create deployment details --image docker.io/istio/examples-bookinfo-details-v1:1.20.2
kubectl get deployments
kubectl get pods
```

```shell
kubectl expose deployment productpage --port 9080 --target-port 9080 --type NodePort
kubectl expose deployment reviews --port 9080 --target-port 9080
kubectl expose deployment details --port 9080 --target-port 9080
kubectl get services
```

http://localhost:32557/productpage


```shell
kubectl get pods
kubectl delete pod productpage-644c575c58-cpxw5
kubectl get pods
```

service hasn't changed, URL still works
notice the name, end of pod name is random string
middle is replicaset

```shell
kubectl get replicaset
```

deployment wraps a replicaset wraps pods
keeps them running
play around deleting 

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
 fails

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

works

Template `v1` to `v2` in two places


