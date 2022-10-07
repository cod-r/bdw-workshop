# Using ArgoCD and Crossplane for Bootstrapping K8s Clusters

## Prerequisites
- [docker](https://docs.docker.com/engine/install/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries)
  - Present alternatives
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [doctl](https://docs.digitalocean.com/reference/doctl/how-to/install)
- [GitHub Account](https://github.com/signup)
- Fork this repository on GitHub: https://github.com/cod-r/bdw-workshop
1. Go to https://github.com
2. Search for `bdw-workshop`
3. Click `cod-r/bdw-workshop`
4. Right upper corner -> Click Fork -> Create Fork
5. Add your Github Username to an environment variable

IMPORTANT
```sh
export GH_USERNAME=<your-gh-username>
```
6. Clone the forked repo
```sh
echo $GH_USERNAME
git clone https://github.com/${GH_USERNAME}/bdw-workshop.git
cd bdw-workshop
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
- Create `argocd` directory
```shell
mkdir argocd
```

- Create ConfigMap
```yaml
cat > argocd/argocd-cm.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
  namespace: argocd
data:
  timeout.reconciliation: 10s
  exec.enabled: "true"
EOF
```
- Apply ConfigMap
```sh
kubectl apply -f argocd/argocd-cm.yaml
```

- Redeploy `argocd-application-controller` for changes to take effect

```sh
kubectl -n argocd rollout restart statefulset argocd-application-controller
```

### Open Argo CD UI
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Password - mai clar
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Open https://localhost:8080

Username: admin  

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
- Commit and push
```sh
git add .
git commit -m "a gitops test"
git push
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

- Commit and push

- Verify
```sh
kubectl get secrets
```

# Chapter 2
Day 2 Operations.

## Install an ingress controller

- Deploy ingress-nginx
```yaml
cat > argocd/applications/ingress-nginx.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/kubernetes/ingress-nginx.git
    path: deploy/static/provider/kind
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
EOF
```

- Commit and push

### Test ingress
- Create ingress for Argo CD
```yaml
cat > argocd/argocd-ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: argocd
  name: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
spec:
  rules:
    - host: localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
EOF
```

- Commit and push
- Access the UI via ingress
https://localhost/

Username: admin  
Password
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## Install monitoring tools

- Deploy kube-prometheus-stack CRDs
```yaml
cat > argocd/applications/kube-prometheus-stack-crds.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  destination:
    server: "https://kubernetes.default.svc"
    namespace: monitoring
  source:
    repoURL: https://github.com/prometheus-community/helm-charts.git
    path: charts/kube-prometheus-stack/crds/
    targetRevision: kube-prometheus-stack-40.3.1
    directory:
      recurse: true
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - Replace=true
EOF
```

- Deploy kube-prometheus-stack
```yaml
cat > argocd/applications/kube-prometheus-stack.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    targetRevision: 40.3.1
    chart: kube-prometheus-stack
    helm:
      skipCrds: true
      values: |
        prometheus:
          prometheusSpec:
            retention: 7d
        grafana:
          grafana.ini:
            server:
              root_url: http://localhost/grafana
              domain: localhost
              serve_from_sub_path: true
          ingress:
            enabled: true
            hosts:
              - localhost
            path: /grafana
          additionalDataSources:
            - name: loki
              type: loki
              url: http://loki-stack.monitoring.svc.cluster.local:3100
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
EOF
```

- Deploy loki-stack
```yaml
cat > argocd/applications/loki-stack.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: loki-stack
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://grafana.github.io/helm-charts
    targetRevision: 2.8.3
    chart: loki-stack
    helm:
      values: |
        loki:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
EOF
```

- Commit and push
- Access Grafana UI
https://localhost/grafana

Username: admin  
Password: prom-operator

## Disaster recovery
- Delete kind cluster
```sh
kind cluster delete
```
- Recreate the cluster
- Install Argo CD in the new cluster
- Cluster recovered


# Chapter 3
## Crossplane

- Deploy crossplane
```yaml
cat > argocd/applications/crossplane.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://charts.crossplane.io/stable
    targetRevision: 1.9.1
    chart: crossplane
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
EOF
```

- Create Application for DigitalOcean provider manifests
```yaml
cat > argocd/applications/crossplane-do.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-do-resources
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/${GH_USERNAME}/bdw-workshop.git
    path: crossplane-do
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-do
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
EOF
```

- Create secret containing the token from DigitalOcean
1. Create token env var
```sh
export DO_TOKEN=<your-do-token>
```
2. Create secret
```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Secret
metadata:
  namespace: crossplane-do
  name: provider-do-secret
type: Opaque
stringData:
  token: ${DO_TOKEN}
```

- Create droplet
```yaml
cat > crossplane-do/droplet.yaml <<EOF
apiVersion: compute.do.crossplane.io/v1alpha1
kind: Droplet
metadata:
  name: ${GH_USERNAME}-crossplane-droplet
  annotations:
    crossplane.io/external-name: crossplane-droplet
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
spec:
  forProvider:
    region: fra1
    size: s-1vcpu-1gb
    image: ubuntu-20-04-x64
  providerConfigRef:
    name: do-config
EOF
```
### Setup doctl
```sh
doctl auth init
```
### Check droplet
```sh
doctl compute droplet list 
```

### Delete droplet
```sh
doctl compute droplet delete crossplane-droplet
```
Wait for droplet to be recreated by Crossplane

- Create k8s cluster
```yaml
cat > crossplane-do/droplet.yaml <<EOF
apiVersion: kubernetes.do.crossplane.io/v1alpha1
kind: DOKubernetesCluster
metadata:
  name: k8s-cluster
  annotations:
    argocd.argoproj.io/sync-wave: "3"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
spec:
  providerConfigRef:
    name: do-config
  forProvider:
    region: fra1
    version: 1.24.4-do.0
    nodePools:
      - size: s-1vcpu-2gb
        count: 1
        name: worker-pool
    maintenancePolicy:
      startTime: "00:00"
      day: wednesday
    autoUpgrade: true
    surgeUpgrade: false
    highlyAvailable: false
EOF
```

- Get kubeconfig
```sh
doctl kubernetes cluster kubeconfig save k8s-cluster
```


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

