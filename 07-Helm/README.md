# Helm

Most of the time we will use `kubectl apply` to declare and implement the
desired state for our cluster. But if we are using third-party code, often we'll
use a package manager called [Helm](https://helm.sh/) to allow for more flexible
deployments in various environments.

Helm is preinstalled, but if you needed to
[install](https://helm.sh/docs/intro/install/) it elsewhere you could install it
with the following.

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Using Helm to install workflow

We will use Helm to install [Kiali](https://kiali.io/), a visualization tool
usually used with [Istio](https://istio.io/).

There are just a few commands we need to install using Helm, but in practice
there are a lot of repeated names in the commands so it might be helpful to
look at them argument-by-argument.

- `helm repo add kiali https://kiali.org/helm-charts`

  This adds the repository to our system so Helm knows about it and the charts
  it contains.

  - `helm repo add`- the command
  - `kiali`- **repo name**
  - `https://kiali.org/helm-charts`- repo URL 

  We only need to do this once, but if we had done it before we might to to get
  the latest lists from all of our installed repositories using 
  `helm repo upgrade`.

- `helm install --namespace istio-system --create-namespace kiali-server kiali/kiali-server`

  This command follows the chart instructions to install an application or
  service or whatever on our system. This command is the whole point of using
  Helm in the first place.

  - `helm install`- the command
  - `--namespace istio-system`- install into the `istio-system` namespace
  - `--create-namespace`- create the `istio-system` namespace if it doesn't exist
  - `kiali-server`- **release name**
  - `kiali/kiali-server`- **repo name**/**chart name**

  This command finishes when the chart is installed. There might be a lot still
  going on behind the scenes, acting on the chart commands. Some systems are
  ready in seconds, some in minutes, and some charts are worth installing just
  before going to lunch.

- `helm status kiali-server --namespace istio-system`

  This looks like a great way to find out what it going on behind the scenes. It
  is not. It is really for reminding us what we did yesterday before we quit for
  the evening, or maybe reminding us if we installed last week or just planned
  to install.

  I only mention it here to save you from false hope when we want to understand
  if the installtion is done.

  - `helm status`- the command
  - `kiali-server`- **release name**
  - `--namespace istio-system`- we installed into a specific namespace, so we need to
    specify that anytime we refer to the release

## Port forwarding

Although we intalled Kiali using Helm to get a chance to try Helm, Kiali itself
is a useful tool.

Kiali is accessable as a web application once it is installed, but as we
remember from [05-Ingress](../05-Ingress/README.md), there are things we have to
do to make it available from outside the cluster. 

Before we get that far, though, we can see if we can access it from within the
cluster. For that, we can try `kubectl port-forward`. 

Open a new terminal using <kbd>Alt</kbd>-<kbd>T</kbd>, and in that window start
the port forwarding. First, we'll need to know where the service is listening.

```shell
kubectl get services -n istio-system
kubectl port-forward service/kiali 15001:20001 -n istio-system
```

That will forward traffic sent to `localhost` on port `15001` to the `kiali`
service on port `20001`. Leave that running. 

Remembering back to our experiences in
[03-Networking](../03-Networking/README.md), we know that means *within the
cluster* includes from our development environment. So back in the original
terminal, see if we can access Kiali from the command line.

```console
$ curl -sS http://localhost:15001/
<a href="/kiali/">Found</a>.
```

Success-ish. Since we don't have access to a UI-based browser here, it doesn't
provide any real value to us other than to prove that we can forward traffic
temporarily without going through the trouble of setting up an ingress. This is
often helpful for troubleshooting or for accessing administration or
configuration consoles on an infrequent basis.

The port forwarding will remain active until we kill it by closing the terminal
or using <kdb>Ctrl</kdb>-<kbd>C</kbd>.

## Kiali

We know how to set up an ingress, though and can add one fairly quickly with
`https` access even. We need `Middleware` for Traefik and a new `Ingress` for
Kiali, and we'll be running it from the `istio-system` namespace. The
`ClusterIssuer` was cluster-wide, so we can reuse that.

Create and apply `kiali-ingress.yaml`, making sure to apply it to the correct
namespace.

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect
  namespace: istio-system
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kiali-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cert
    traefik.ingress.kubernetes.io/router.middlewares: istio-system-redirect@kubernetescrd
spec:
  rules:
  - host: william.codemash.otherdevopsgene.dev # replace william with your username
    # or william.codemash.otherdevopsgene.work or william.codemash.otherdevopsgene.xyz
    http:
      paths:
      - path: /kiali
        pathType: Prefix
        backend:
          service:
            name: kiali
            port:
              number: 20001
  tls:
  - hosts:
    - william.codemash.otherdevopsgene.dev # match the host above
    secretName: acme-tls-cert
```

```shell
kubectl apply -f kiali-ingress.yaml -n istio-system
```

This manifest file is different than the others we've created and looked at. It
have multiple YAML documents, separated by three dashes (`---`) that signal the
start of each. Technically, we could have been leading off our manifests with
them, but they haven't been needed until now. To be honest, we could skip the
first separator now, too, but it helps remind m that this is a multiple document
YAML file.

Next, we added the namespace to the `kubectl apply` command. We could have used
`--namespace istio-system` to be clearer, but namespaces are so common that we
almost always shorten it to save typing.

Along with that, we can see all the ingresses (both of them) across the
namespaces by querying `--all-namespaces`, or `-A` for short.

```shell
kubectl get ingress -A
```

In any case, we can now reach Kiali from our laptop at
`https://william.codemash.otherdevopsgene.dev/kiali`, replacing `william` with
your username as we did earlier with our first ingress and using `.work` or
`.xyz` as the top-level domain if we didn't use `.dev`.

## Creating a token

And we are stopped at the front door by Kiali asking for a token, aka a service
account token. That is a string representing us as a particular user, or service
account. Once we prove who we are, Kiali will let us in.

The Kiali documentation describes [how to obtain a token when logging in via
token auth
strategy](https://kiali.io/docs/faq/authentication/#how-to-obtain-a-token-when-logging-in-via-token-auth-strategy).
And when we try that, we see an error.

```console
$ kubectl -n istio-system create token kiali-service-account
error: failed to create token: serviceaccounts "kiali-service-account" not found
```

*Note*: the namespace can go almost anywhere in the command. Earlier is often
better for command completion.

So the docs and the Helm install aren't 100% in sync. The crux is that we need
to know the name of the service account for the token. But there is enough
context in the error message to point us in the right direction.

```shell
kubectl -n istio-system get serviceaccounts
```

And from the output of that, we can guess that the service account name is
simply `kiali`. Therefore the token can be created with:

```shell
kubectl -n istio-system create token kiali
```

Using that long string gets us into Kiali, although there are a few scattered
errors about `prometheus.istio-system` not being found.

[Prometheus](https://prometheus.io/) is a very popular metrics tool. We can look
for a Helm chart to install that as well by searching Google or search the
Artifact Hub for Helm.

```shell
helm search hub prometheus
```

That returns us way more options than we could sort through. The frst one
happens to be the one we want, but we also need the URL for the repo so we can
add it.

```console
$ helm search hub prometheus --list-repo-url | head -2
URL                                                     CHART VERSION   APP VERSION             DESCRIPTION                                             REPO URL                                          
https://artifacthub.io/packages/helm/prometheus...      26.1.0          v3.1.0                  Prometheus is a monitoring system and time seri...      https://prometheus-community.github.io/helm-charts
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
```

Now we can see the repository has been added and then we can search the
`prometheus-community` repo for the right chart.

```shell
helm repo list
helm search repo prometheus-community
```

Again, an unwieldy list. In many cases, searching with Google will give more
information about choosing the appropriate Helm charts than Helm can itself. In
our case, we need the `prometheus` chart (from the `prometheus-community` repo).

## Exercise

1. Use Helm to install Prometheus in the `istio-system` namespace. 
2. Update our `kiali-ingress` to send `/prometheus` traffic to port `80` on
   the `prometheus` service.

<!-- helm install prometheus prometheus-community/prometheus -n istio-system -->

<!--  
      - path: /prometheus
        pathType: Prefix
        backend:
          service:
            name: prometheus-server
            port:
              number: 80
-->

## Aftermath

While good practice using Helm, our little cluster isn't big enough to handle
everything we've thrown at it. Many of the pods we are using are now `Evicted`,
kicked out for something the Kubernetes scheduler thinks is more important.

Fixing the Prometheus installation would be the right way forward, but we would
need a bigger cluster to do that. Instead, let's use Helm to undo the damage by
uninstalling Prometheus.

```shell
helm uninstall prometheus -n istio-system
```

We've overwhelmed our cluster so much that it won't fix itself, at least not
right away. But this makes a good stopping point.

## End of lesson
