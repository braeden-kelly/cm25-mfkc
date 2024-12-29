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

```shell
docker rm -f $(docker ps -q)
docker ps
```

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

Create `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookinfo-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: productpage
            port:
              number: 9080
```

```shell
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress bookinfo-ingress
```

`traefik` is deployed with k3s. An ingress maps to a service.

`curl https://172.31.14.81/productpage`

works even from your neighbor's environment. Not from your laptop since k3s only
knows about the private IP address. But if you had a public IP address (you do),
and the firewalls were open (they are), you could use the public IP.

AWS Sidebar:

`curl http://checkip.amazonaws.com` or `curl icanhazip.com`

Right way, use the AWS Instance Metadata Service:

```shell
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
```

Visit `http://3.147.27.216/productpage` from your local browser.

Because Kubernetes is a well-defined interface, people can write other code that
works with that framework and API and adds significant capability, without
having to know which implementation of Kubernetes you are using (usually).

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
kubectl get services
kubectl get namespaces
kubectl get pods --namespace cert-manager
kubectl get pods -A
kubectl logs -n kube-system traefik-57b79cf995-dgc77
```

Let's Encrypt aka ACME

`cluster-issuer.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cert
spec:
  acme:
    email: letsencrypt@otherdevopsgene.dev #Replace with your email
    server: https://acme-staging-v02.api.letsencrypt.org/directory # https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-cert-private-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

```shell
kubectl apply -f cluster-issuer.yaml
kubectl get secrets -n cert-manager
kubectl describe secret -n cert-manager letsencrypt-cert-private-key
```

Not actually secret

```shell
kubectl get secret -n cert-manager letsencrypt-cert-private-key --template="{{index .data \"tls.key\"}}" | base64 -d
kubectl get secret -n cert-manager letsencrypt-cert-private-key -o json | jq '.data | map_values(@base64d)'
```

JSON, YAML, [Go Templates](https://pkg.go.dev/text/template), JSON Path

Could have used `--template={{.data.tls.key}}` except `tls.key` has a dot.

`ConfigSet` is the same thing, without pretending to be secret. Just name-value
pair store.

Update `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookinfo-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cert # name from cluster-issuer
spec:
  rules:
  - host: tom.codemash.otherdevopsgene.dev # replace tom with your username
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: productpage
            port:
              number: 9080
  tls:
  - hosts:
    - tom.codemash.otherdevopsgene.dev # replace tom with your username
    secretName: acme-tls-cert
```

```shell
kubectl apply -f ingress.yaml
kubectl describe ingress bookinfo-ingress
```

TLS and CreateCertificate

```shell
kubectl get secrets
kubectl describe secrets acme-tls-cert
kubectl get ingress
kubectl describe ingress cm-acme-http-solver-bfm2g
```

Understated value of Kubernetes. Installed cert-manager, pointed it to Let's
Encrypt. Redeployed our ingress to include a hostname and the cluster-issuer ==
cert-manager configuration we wanted to use.

We proved that we were entitled to use the domain name
(cm-acme-http-solver-bfm2g), generated a certificate request, requested a cert
with the CA, received and installed the certificate, set up the Ingress for TLS
termination. All that's left is to make `http` redirect to `https`.

`curl --head http://bethtruss.codemash.otherdevopsgene.dev/productpage`

Notice the `200 OK`

This step happens to be Traefik-specific.

`middleware.yaml`

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

In `ingress.yaml` add an annotation:

```yaml
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd
```

`namespace`-`middlewarename`@kubernetescrd

```shell
kubectl apply -f middleware.yaml
kubectl apply -f ingress.yaml
```

`curl --head http://bethtruss.codemash.otherdevopsgene.dev/productpage`

Notice the `308 Permanent Redirect`

`curl --head https://bethtruss.codemash.otherdevopsgene.dev/productpage`

Deployment rollout, CrashLoopBackOff
readiness-liveness
Volumes
Namespaces - Multi-tenant

Demo: AWS EKS, autoscaling, Cilium, Hubble, Network Policies

Template `v1` to `v2` in two places

For admins, do [Kubernetes the Hard
Way](https://github.com/kelseyhightower/kubernetes-the-hard-way). Then never do
it without a tool again.

For practice,

- [Tutorials](https://kubernetes.io/docs/tutorials/)
- [KodeKloud Kubernetes Free Labs](https://kodekloud.com/free-labs/kubernetes).

More learning,

- [LinuxFoundationX: Introduction to
  Kubernetes](https://www.edx.org/learn/kubernetes/the-linux-foundation-introduction-to-kubernetes)
- [TechWorld with Nana](https://youtu.be/s_o8dwzRlu4?si=T38bQK3OhwbIEzJ7)
