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

## Install Argo CD

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
## Make Argo CD manage itself

1. Apply the main app that will manage all files in this repository
```shell
kubectl apply -f argocd/applications/main-app.yaml
```
After applying the manifest Argo CD will manage itself. 
Every push to this repository will be applied automatically by Argo CD.

2. Access the UI
https://localhost/

Username: admin  
Password
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```


## Install Argo CD
- [Argo CD initial setup](argocd/initial-setup.md)

## Install Crossplane
- Crossplane is installed automatically by Argo CD. See [crossplane.yaml](argocd/applications/crossplane.yaml)
