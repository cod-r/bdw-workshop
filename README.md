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
Add env to argocd-application-controller:
```yaml
    env:
      - name: "ARGOCD_SYNC_WAVE_DELAY"
        value: "30"
```

### Change refresh interval
```sh
kubectl apply -f argocd/argocd-cm.yaml
```
Redeploy `argocd-application-controller` for changes to take effect

```sh
kubectl -n argocd rollout restart statefulset argocd-application-controller
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


## GKE
- Get kubeconfig
```sh
gcloud container clusters get-credentials gke-cluster
```

- create service account and clusterrolebinding on the destination cluster:

```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: argocd
  namespace: kube-system
EOF
```

- get server address

```sh
kubectl cluster-info
```

- get certificate and the token

```sh
TOKEN_SECRET=$(kubectl -n kube-system get sa argocd \
    -o go-template='{{range .secrets}}{{.name}}{{"\n"}}{{end}}')
CA_CRT=$(kubectl -n kube-system get secrets ${TOKEN_SECRET} -o go-template='{{index .data "ca.crt"}}')
TOKEN=$(kubectl -n kube-system get secrets ${TOKEN_SECRET} -o go-template='{{.data.token}}' | base64 -d)
```

- create cluster secret in kind cluster

```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gke-cluster-conn-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: gke-cluster
  server: https://34.118.116.43
  config: |
    {
      "bearerToken": "${TOKEN}",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "${CA_CRT}"
      }
    }
EOF
```

