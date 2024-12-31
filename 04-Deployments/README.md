# Deployments



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
kubectl expose deployment productpage --port 9080 --target-port 9080
kubectl expose deployment reviews --port 9080 --target-port 9080
kubectl expose deployment details --port 9080 --target-port 9080
kubectl get services
```

http://10.43.194.60:9080/productpage

```shell
kubectl get pods
kubectl delete pod productpage-644c575c58-cpxw5
kubectl get pods
kubectl get services
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
kubectl scale deployment ...
```

