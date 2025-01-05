# Ingress

When we looked at networking before, we focused on networking within the cluster
opening a view into an application that might be suited for development or
debugging, since you need to know which node the application was running on.

Now, we are going to look at something that might be appropriate for production. 

## Load balancers

**Spoiler**: We aren't going to spend any time on load balancers. Although they are
truly the next level of networking in the Kubernetes world, after `ClusterIP` and
`NodePort`, they are often implemented outside the cluster itself. That presents
some logistical challenges, and potentially cost, in a workshop with many
students.

They work operate very much like traditional load balancers, and in
Kubernetes they also can be used to make an application within the cluster
accessible to the world outside. 

In turn, they have a significant overlap with an `Ingress`, which also can be
used to make an application accisible from outside the cluster. And an ingress
is often implemented within the cluster. For this reason, ingresses are a
popular alternativeto load balancers in the Kubernetes world.

## Ingress

An ingress provides some simple routing to a service.

Create and apply `ingress.yaml`, and then examine what it produces.

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
kubectl get ingresses
kubectl describe ingress bookinfo-ingress
```

Note that the class is listed as `traefik`, which is an ingress controller that
is distributed with `k3s`. The ingress is assigned the IP address of the system
we are using, albeit listed as the private IP. 

As configured, all traffic (path prefix '/') is sent to port 9080 on the
`productpage` service. We can try it.

```shell
curl -sS http://${PRIVATE_IPV4}/productpage
```

Like the NodePort configuration, this works even from our neighbor's
environment. It won't work as is from your laptop since `k3s` only
knows about the private IP address. But if we had a public IP address (we do),
and the firewalls were open (they are), we could use the public IP.

```shell
curl -sS http://${PUBLIC_IPV4}/productpage
```

If we use the value for the public IP address, it works from the command line on
our laptops. Or, we can visit the same URL from a laptop browser.

On the dev environment:

```shell
echo ${PUBLIC_IPV4}
```

Then on the laptop, use `http://111.222.333.444/productpage`, replacing
`111.222.333.444` with the value of our public IP.

Back in the first lesson, we discussed that could use a domain name if we mapped
it to the public IP address (which we did). Try
`http://lanemeyer.codemash.otherdevopsgene.dev/productpage`. Replace
`lanemeyer` with the username you used to login to AWS. If you get an error, your
browser is probably protecting you by *fixing* the URL to use `https`. 

Turns out, we can make that work, too.

## Namespaces

Because Kubernetes is a well-defined interface, people can write code that
works with that framework and API and adds significant capability, without
having to know which implementation of Kubernetes you are using (usually).

Also, `kubectl apply` can take a URL rather than just a filename. Since the
declarative form doesn't expect a certain starting state, we can use a manifest
for a third-party plugin to install itself on our cluster.

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```

That isn't a single resource, but a file with many resources in it. A lot of the
types are new to us, but there are some services at the end, so let's start by
looking at them.

```shell
kubectl get services
```

Probably not what we expected. The `kubectl apply` said it created
`cert-manager-cainjector`, `cert-manager`, and `cert-manager-webhook`, but they
aren't there. So where did they go?

Well, the first line of the `kubectl apply` said it created
`namespace/cert-manager`, so let's try `namespaces`. 

```shell
kubectl get namespaces
```

We can see there are a few, most of which have been around since we started
(based on their age), and then the one new namespace called `cert-manager`.

Namespaces are a way to logically segment applications and resources. Until now,
we've been working in the default namespace, conventiently called `default`. But
almost every command we've used so far can also take a `--namespace` option.

```shell
kubectl get services,deployments,pods --namespace cert-manager
```

We could also look across all namespaces with the `--all-namespaces` or `-A`
option.

```shell
kubectl get pods -A
```

We can see the new pods that `cert-manager` installed, the pods in the `default`
namespace that we've been creating, and some parts of the system that `k3s`
brought in to start in the `kube-system` namespace, including the `traefik`
ingress controller we worked with above.

Now we need a certificate. And we'll see some new resource types on the way.

Create and apply `cluster-issuer.yaml`, making sure to replace the email address
appropriately.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cert
spec:
  acme:
    email: otherdevopsgene@portinfo.com # Replace with your email
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-cert-private-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

Notice from the `apiVersion` that this is not a standard Kubernetes resource.
This is a custom resource that `cert-manager` defined, specifically
`customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io`.
Custom resource definitions, or CRDs, are a way to extend the Kubernetes API. We
won't get into writing a CRD, but obviously we are starting to use them.

```shell
kubectl apply -f cluster-issuer.yaml
kubectl get clusterissuer
kubectl describe clusterissuer
```

The `ClusterIssuer` doesn't live in a namespace because it is cluster-wide.
There is a version called an `Issuer` that can be namespace-specific.

Aside from the information we supplied, we can see some status that says we
registered with the ACME server, which is the certificate authority called
[Let's Encrypt](https://letsencrypt.org/).

## Secrets

Some other resouces got created as well. Let's look into one in particular.

```shell
kubectl get secrets -n cert-manager
kubectl describe secret -n cert-manager letsencrypt-cert-private-key
```

It is called a secret and has one piece of data (therefore, technically a datum)
called `tls.key`. If you know anything about how certificates work, you might
recognize that as the private key that keeps a certificate secure.
Unfortunately, despite the name, a Kubernetes `secret` is not actually secret,
at least not by default. It is just a collection of Base64-encoded name-value
pairs.

Try either (or both) of:

```shell
kubectl get secret -n cert-manager letsencrypt-cert-private-key --template="{{index .data \"tls.key\"}}" | base64 -d
kubectl get secret -n cert-manager letsencrypt-cert-private-key -o json | jq '.data | map_values(@base64d)'
```

The complexity around the Go template is because the single name-value pair
`tls.key` has a dot in the name, so `--template={{.data.tls.key}}` would look
like an attribute named `key` further in the hierarchy. 

But the point is that this secret is just encoded, not encrypted, so is obscured
from casual viewing, but not secret at all. There are ways to change that, but
we won't have time to explore further. Just remember, without some changes, a
`secret` isn't secret.

A `ConfigSet` is almost the same thing, without pretending to be secret. Just a
name-value pair store. We'll play with that resource later.

## TLS traffic

Update and apply `ingress.yaml` to know about our `cert-manager` and more
importantly our `ClusterIssuer` which will handle getting certificates from
Let's Encrypt for us. The ingress also needs to know what name to 

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookinfo-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cert # name from cluster-issuer
spec:
  rules:
  - host: lanemeyer.codemash.otherdevopsgene.dev # replace lanemeyer with your username
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
    - lanemeyer.codemash.otherdevopsgene.dev # replace lanemeyer with your username
    - 111.222.333.444 # replace with your public IP
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

`curl --head http://lanemeyer.codemash.otherdevopsgene.dev/productpage`

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

`curl --head http://lanemeyer.codemash.otherdevopsgene.dev/productpage`

Notice the `308 Permanent Redirect`

`curl --head https://lanemeyer.codemash.otherdevopsgene.dev/productpage`


Let's see what it logged using the `kubectl logs` command. Replace
`traefik-57b79cf995-dgc77` with the name of the `traefik` pod on your cluster.

```shell
kubectl logs -n kube-system traefik-57b79cf995-dgc77
```