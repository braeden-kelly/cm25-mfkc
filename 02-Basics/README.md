# Basics

## Cluster

We will start with a cluster already available to us. (If that isn't the case,
stand one up.)

We will use the simple but full-featured [k3s](https://k3s.io/). We can tell
that it running by accessing the Kubernetes API through our soon-to-be favorite
tool, `kubectl`.

```shell
kubectl cluster-info
```

Technically, `kubectl` just made an API call to the configured cluster and
parsed and displayed the response. That gives us some evidence the cluster is
running and accessible to us.

We can see a little more by checking out the nodes that are part of our cluster,
in this case just a single node.

```shell
kubectl get nodes
```

Still not much to see, but this does introduce one of the most common commands
we'll use, `kubectl get` *resource_type*. That provides us a list of all the
resources of the requested type with some basic information. There are caveats
to it, but that is enough to understand for now.

Now let's get something running.

## Pods

Pods are a wrapper around one or more container images that are deployed and
managed together, very often just one. They are one of the b

Currently, we have not started any pods. 

```shell
kubectl get pods
```

We can run the same **Bookinfo** microservices application as before, but this
time using Kubernetes. 

![Bookinfo](../bookinfo-basic.svg)
<img src="../bookinfo-basic.svg">

As a reminder, requests come into `productpage`. That service makes requests to
`reviews` and `details` to gather the information to display. 

```shell
kubectl run productpage --image docker.io/istio/examples-bookinfo-productpage-v1:1.20.2
kubectl run reviews --image docker.io/istio/examples-bookinfo-reviews-v1:1.20.2
kubectl run details --image docker.io/istio/examples-bookinfo-details-v1:1.20.2
```

That isn't significantly different than starting the containers with Docker. And
it isn't any more functional. These are just independently running containers
that aren't specifically configured to find and talk to each other at this
point.

We can see what we've started by getting the list of the running pods.

```shell
kubectl get pods
kubectl get pods -o wide
```

Notice that the cluster has assigned an internal IP address to each of the pods.

Adding the output format (`-o wide`) gets us a little more information about
each pod. 
