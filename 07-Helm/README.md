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

## Helm installation workflow

helm repo add kiali https://kiali.org/helm-charts
helm install --namespace kiali --create-namespace kiali-server kiali/kiali-server

