# bdw-workshop

## Install kind

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

## Create kind cluster

```sh
kind create cluster --config kind-config.yaml
```

## Install ingress-nginx
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```
- Test ingress
```sh
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/usage.yaml
```

```sh
curl localhost/foo
```

## Install Argo CD
- [Argo CD initial setup](argocd/initial-setup.md)

## Install Crossplane
- Crossplane is installed automatically by Argo CD. See [crossplane.yaml](argocd/applications/crossplane.yaml)
