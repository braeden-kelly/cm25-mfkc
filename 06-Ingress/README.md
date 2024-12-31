# Ingress and cert-manager

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
