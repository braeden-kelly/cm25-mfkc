# Networking

To get our services talking to each other, we need to expose the ports they will
use to communicate.

```shell
kubectl expose pod productpage --port 9080 --target-port 9080
kubectl expose pod reviews --port 9080 --target-port 9080
kubectl expose pod details --port 9080 --target-port 9080
```

The `--port` option tells the cluster what port number we will be using, and the
`--target-port` specifies the port on the container it will be mapped to. Since
Kubernetes has assigned a separate IP address for each pod, we can use the same
port number for each of the services.

BTW, this is known as the *imperative* form of creating a service, in that we
tell the cluster what we want done. It is useful on occasion, for learning, and
on certification exams, but we'll get out of the habit soon enough. Then we will
move to the *declarative* form, where we declare what we want the end result to
be and Kubernetes makes the necessary adjustments to get us there.

Let's see the services we created.

```shell
kubectl get services
```

Cluster IP

If we don't want the entire list of resources from the `kubectl get`
*resource_type* command, we can be more specific and ask for a particular
*resource_name*.

```shell
kubectl get services productpage
```



`curl http://10.99.151.9:9080/productpage` works, clusterip
internal only, since we are hosting the cluster on the same box
NodePort maps a high port 30000–32768 from the cluster itself to a service

```shell
kubectl delete service productpage
kubectl expose pod productpage --port 9080 --target-port 9080 --type NodePort
kubectl get services
```

http://localhost:32746/productpage

ClusterIp exposure < NodePort exposure < LoadBalancer exposure

ClusterIp
Expose service through k8s cluster with ip/name:port
NodePort
Expose service through Internal network VM's also external to k8s ip/name:port
Ingress
Expose service through External world.

```shell
curl http://10.43.179.185:9080/reviews/2
curl http://reviews.default.svc.cluster.local:9080/reviews/2
```

port-forwarding


```shell
kubectl delete service details productpage reviews
kubectl delete pod details productpage reviews
```
