# Using ArgoCD and Crossplane for Bootstrapping K8s Clusters

## Prerequisites
- Access https://github.com/cod-r/bdw-workshop
- [docker](https://docs.docker.com/engine/install/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries)
  - Or any other kubernetes cluster (Rancher Desktop, k3d, minikube etc.)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [GitHub Account](https://github.com/signup)
- [Add your ssh key to your GitHub account](github-ssh.md)
- Fork this repository on GitHub: https://github.com/cod-r/bdw-workshop
1. Click Fork (right upper corner) -> Create Fork
2. Add your Github Username to an environment variable

**IMPORTANT:** This variable will be used in the next steps.  
Don't miss this step!
```sh
GH_USERNAME=<your-gh-username>
```
3. Clone the forked repo
```sh
echo $GH_USERNAME
git clone git@github.com:${GH_USERNAME}/bdw-workshop.git
cd bdw-workshop
``` 

# Chapter 1
Initial Argo CD setup
## Create kind cluster
Skip this step if you already have a cluster.
```sh
kind create cluster --config kind-config.yaml
```

## Install Argo CD
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Access Argo CD UI
```sh
kubectl port-forward svc/argocd-server -n argocd 8083:443
```

Open https://localhost:8083

Username: admin  
Password: `run the command below to print the password`
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## Make Argo CD manage itself

### Create the main Application.  
This app will apply all manifests found in `argocd/applications` directory in this repository.

1. Create directories
```sh
mkdir -p argocd/applications
```

2. Create `main-app.yaml` file
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

3. Apply the created manifest
```sh
kubectl apply -f argocd/applications/main-app.yaml
```

After applying the manifest we can add other manifests in `argocd/applications` and Argo CD will apply them automatically.

### Test GitOps
1. Create a simple pod
```yaml
cat > argocd/applications/test-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.23.1
EOF
```

2. Commit and push
```sh
git add . && git commit  -m "argocd setup test" && git push
```

3. Check created pod
```sh
kubectl get pods
```

# Chapter 2
Day 2 Operations

## Install monitoring tools

1. Add kube-prometheus-stack CRDs
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

2. Add kube-prometheus-stack
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

3. Add loki-stack
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

4. Commit and push
```sh
git add . && git commit  -m "deploy kube-prometheus-stack and loki" && git push
```

### Access Grafana
```sh
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```
Open http://localhost:3000

Username: admin  
Password: prom-operator

## Disaster recovery
1. Delete kind cluster
```sh
kind delete cluster
```
2. Recreate the cluster
```sh
kind create cluster --config kind-config.yaml
```

3. Install Argo CD in the new cluster
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

4. Apply the `main-app` manifest
```sh
kubectl apply -f argocd/applications/main-app.yaml
```

5. Print the new password
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Cluster recovered!


# Chapter 3
## Crossplane

1. Deploy crossplane
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
git add . && git commit  -m "deploy crossplane" && git push
```

2. Create Application for DigitalOcean provider manifests
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
git add . && git commit  -m "add crossplane digitalocean provider" && git push
```

3. Create an env var with lowercase letters only
```sh
LC_USER=$(echo "$GH_USERNAME" | tr '[:upper:]' '[:lower:]')
echo $LC_USER # must be lowercase
```

4. Create droplet  
A droplet is a DigitalOcean VM.
```yaml
cat > crossplane-do/droplet.yaml <<EOF
apiVersion: compute.do.crossplane.io/v1alpha1
kind: Droplet
metadata:
  name: ${LC_USER}-crossplane-droplet
  annotations:
    crossplane.io/external-name: ${LC_USER}-crossplane-droplet
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
spec:
  forProvider:
    region: fra1
    size: s-1vcpu-1gb
    image: ubuntu-20-04-x64
  providerConfigRef:
    name: do-config
EOF
git add . && git commit  -m "create digitalocean droplet via crossplane" && git push
```
Droplet creation will fail because we don't have credentials to access DigitalOcean.

5. Create a Secret containing the access token from DigitalOcean

- Access https://shorturl.at/fiKQ7
- Create token env var
```sh
DO_TOKEN=<your-do-token>
```

- Create secret
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
EOF
```

6. List droplets
```sh
docker run -it -e DIGITALOCEAN_ACCESS_TOKEN=${DO_TOKEN} \
digitalocean/doctl compute droplet list 
```

7. Delete droplet
```sh
docker run -it -e DIGITALOCEAN_ACCESS_TOKEN=${DO_TOKEN} \
digitalocean/doctl compute droplet delete ${LC_USER}-crossplane-droplet
```
Wait for droplet to be recreated by Crossplane.

8. Delete droplet using Crossplane
```sh
rm crossplane-do/droplet.yaml
git add . && git commit  -m "delete digitalocean droplet" && git push
```

### Create a k8s cluster
```yaml
cat > crossplane-do/k8s-cluster.yaml <<EOF
apiVersion: kubernetes.do.crossplane.io/v1alpha1
kind: DOKubernetesCluster
metadata:
  name: ${LC_USER}-k8s-cluster
  annotations:
    argocd.argoproj.io/sync-wave: "3"
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
spec:
  providerConfigRef:
    name: do-config
  forProvider:
    region: fra1
    version: 1.23.10-do.0
    nodePools:
      - size: s-1vcpu-2gb
        count: 1
        name: ${LC_USER}-worker-pool
    maintenancePolicy:
      startTime: "00:00"
      day: wednesday
    autoUpgrade: true
    surgeUpgrade: false
    highlyAvailable: false
EOF
git add . && git commit  -m "create digitalocean k8s cluster via crossplane" && git push
```

1. List clusters
```sh
docker run -it -e DIGITALOCEAN_ACCESS_TOKEN=${DO_TOKEN} \
digitalocean/doctl kubernetes cluster list
```

2. Save the new cluster kubeconfig
```sh
docker run -it -e DIGITALOCEAN_ACCESS_TOKEN=${DO_TOKEN} \
-v ~/.kube:/root/.kube \
digitalocean/doctl kubernetes cluster kubeconfig save --expiry-seconds 3600 ${LC_USER}-k8s-cluster
```
The kubectl context will change.

3. Check cluster connection
```sh
kubectl get nodes
```

# Chapter 4
Bootstrapping external clusters

## Connect DigitalOcean external cluster to Argo CD

1. Create a serviceaccount and clusterrolebinding on the destination cluster
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

2. Get server address, certificate and the token from external cluster
```sh
CLUSTER_SERVER_ADDRESS=$(kubectl config view --minify -o jsonpath='{.clusters[].cluster.server}')
TOKEN_SECRET=$(kubectl -n kube-system get sa argocd -o go-template='{{range .secrets}}{{.name}}{{"\n"}}{{end}}')
CA_CRT=$(kubectl -n kube-system get secrets ${TOKEN_SECRET} -o go-template='{{index .data "ca.crt"}}')
TOKEN=$(kubectl -n kube-system get secrets ${TOKEN_SECRET} -o go-template='{{.data.token}}' | base64 -d)

echo $CLUSTER_SERVER_ADDRESS
echo $TOKEN_SECRET
echo $CA_CRT
echo $TOKEN

```

3. Switch kubectl context to the cluster where Argo CD is installed  
For kind CONTEXT_NAME is `kind-kind`.
```sh
kubectl config use-context CONTEXT_NAME
```

4. Create Argo CD cluster secret
```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Secret
metadata:
  name: do-cluster-conn-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: do-cluster
  server: ${CLUSTER_SERVER_ADDRESS}
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

## Bootstrap the new cluster

1. Add kube-prometheus-stack CRDs
```yaml
cat > argocd/applications/do-kube-prometheus-stack-crds.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: do-kube-prometheus-stack-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  destination:
    server: ${CLUSTER_SERVER_ADDRESS}
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

2. Add kube-prometheus-stack
```yaml
cat > argocd/applications/do-kube-prometheus-stack.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: do-kube-prometheus-stack
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
          additionalDataSources:
            - name: loki
              type: loki
              url: http://loki-stack.monitoring.svc.cluster.local:3100
  destination:
    server: ${CLUSTER_SERVER_ADDRESS}
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

3. Add loki-stack
```yaml
cat > argocd/applications/do-loki-stack.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: do-loki-stack
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
    server: ${CLUSTER_SERVER_ADDRESS}
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

4. Commit and push
```sh
git add . && git commit  -m "deploy kube-prometheus-stack and loki in digitalocean cluster" && git push
```

5. Check the deployed apps in the new cluster
```sh
kubectl config use-context do-fra1-${LC_USER}-k8s-cluster
kubectl get pods -n monitoring
```

## Google Cloud Platform

1. Configure your GCP account to be ready for integration with Crossplane

See [official documentation](https://github.com/crossplane/crossplane.github.io/blob/master/content/docs/v1.9/cloud-providers/gcp/gcp-provider.md)

2. Create Application for Google Cloud provider manifests
```yaml
cat > argocd/applications/crossplane-gcp.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-gcp-resources
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/${GH_USERNAME}/bdw-workshop.git
    path: crossplane-gcp
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-gcp
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
EOF
git add . && git commit  -m "add crossplane google cloud provider" && git push
```

3. Create ProviderConfig secret

4. Check if the cluster has been created
```sh
gcloud container clusters list
```

5. Save the new GKE cluster kubeconfig
```sh
gcloud container clusters get-credentials gke-cluster
```

## Connect GKE external cluster to Argo CD

1. Create a serviceaccount and clusterrolebinding on the GKE cluster
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

2. Get server address, certificate and the token from GKE cluster

```sh
CLUSTER_SERVER_ADDRESS=$(kubectl config view --minify -o jsonpath='{.clusters[].cluster.server}')
TOKEN_SECRET=$(kubectl -n kube-system get sa argocd -o go-template='{{range .secrets}}{{.name}}{{"\n"}}{{end}}')
CA_CRT=$(kubectl -n kube-system get secrets ${TOKEN_SECRET} -o go-template='{{index .data "ca.crt"}}')
TOKEN=$(kubectl -n kube-system get secrets ${TOKEN_SECRET} -o go-template='{{.data.token}}' | base64 -d)

echo $CLUSTER_SERVER_ADDRESS
echo $TOKEN_SECRET
echo $CA_CRT
echo $TOKEN
```

3. Switch kubectl context to the cluster where Argo CD is installed  
```sh
kubectl config use-context CONTEXT_NAME
```

4. Create Argo CD cluster secret
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
  server: ${CLUSTER_SERVER_ADDRESS}
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

### Deploy a pod in the new cluster via Argo CD
1. Create directory for GKE cluster manifests
```sh
mkdir -p clusters/gke-cluster
```

2. Create Pod manifest
```yaml
cat > clusters/gke-cluster/test-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.23.1
EOF
```

3. Create Argo CD app to apply the manifests in the new cluster
```yaml
cat > argocd/applications/gke-cluster-manifests.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gke-cluster-manifests
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/${GH_USERNAME}/bdw-workshop.git
    path: clusters/gke-cluster
  destination:
    server: ${CLUSTER_SERVER_ADDRESS}
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true 
      allowEmpty: false 
    syncOptions:
      - CreateNamespace=true
EOF
git add . && git commit  -m "Create app to manage GKE cluster" && git push
```

4. Check the deployed apps in the new cluster
```sh
kubectl config use-context GKE_CLUSTER_CONTEXT
kubectl get pods
```