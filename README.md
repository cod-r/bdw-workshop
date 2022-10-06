# Using ArgoCD and Crossplane for Bootstrapping K8s Clusters

## Prerequisites
- [docker engine](https://docs.docker.com/engine/install/)
- [GitHub Account](https://github.com/signup)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- Fork the repository on GitHub: https://github.com/cod-r/bdw-workshop
1. Go to https://github.com
2. Search for `bdw-workshop`
3. Click `cod-r/bdw-workshop`
4. Click Fork -> Create Fork
5. Clone the forked repo
```sh
export GH_USERNAME=<your-gh-username>
echo $GH_USERNAME

git clone https://github.com/${GH_USERNAME}/bdw-workshop.git
```

# Chapter 1

## Create kind cluster

```sh
kind create cluster --config kind-config.yaml
```

## Install Argo CD

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Change refresh interval
```sh
kubectl apply -f argocd/argocd-cm.yaml
```
Redeploy `argocd-application-controller` for changes to take effect

```sh
kubectl -n argocd rollout restart statefulset argocd-application-controller
```

### Open Argo CD UI
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080

Username: admin  
Password
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## Make Argo CD manage itself

1. Create the main Application.  
This app will apply all manifests found in `argocd/applications` directory in this repository.

- Create directories
```shell
mkdir -p argocd/applications
```

- Create `main-app.yaml` file
```yaml
cat > argocd/applications/main-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: main-argocd-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/${GH_USERNAME}/bdw-workshop.git
    path: argocd
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
EOF
```

- Apply the manifest
```sh
kubectl apply -f argocd/applications/main-app.yaml
```

After applying the manifest we can add other manifests in `argocd/applications` and Argo CD will apply them automatically.

### Test GitOps
- Create a simple secret manifest
```yaml
cat > argocd/applications/test.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
stringData:
  my-key: my-value
EOF
```

- Push in git repository
```sh
git add .
git commit -m "a gitops test"
git push
```

## Chapter 2
Day 2 Operations.


## Install ingress-nginx
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

