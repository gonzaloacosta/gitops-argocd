# ArgoCD

### Install ArgoCD in Minikube

- Export variables

```
export INGRESS_HOST=(minikube ip)
```

- Create install variables for ArgoCD Helm Charts

```
helm repo add argo https://argoproj.github.io/argo-helm

cat << EOF >> argocd/argocd-values.yaml
server:
  ingress:
    enabled: true
  extraArgs:
  - --insecure
installCRDs: false
EOF
``` 

- Create namespace and install chart

```
kubectl create namespace argocd

helm upgrade --install argocd argo/argo-cd --namespace argocd --version 2.8.0 --set server.ingress.hosts="{argocd.$INGRESS_HOST.nip.io}" --values argo/argocd-values.yaml --wait
```

- Login in ArgoCD

```
export PASS=$(kubectl --namespace argocd get pods --selector app.kubernetes.io/name=argocd-server --output name | cut -d'/' -f 2)
argocd login --insecure --username admin --password $PASS --grpc-web argocd.$INGRESS_HOST.nip.io
argocd account update-password
open http://argocd.$INGRESS_HOST.nip.io
```

## Deploy an ArgoCD Application from CLI

- Check projects, all application by default is saved in default argocd projects resource

```
argocd proj list
kubectl create namespace voting-app
```

- Create argocd application resource

```
argocd app create voting-app --repo https://github.com/gonzaloacosta/voting-app.git --path "./helm" --dest-server https://kubernetes.default.svc --dest-namespace voting-app
```

- Check app and get k8s objects

```
kubectl -n voting-app get all
argocd app sync voting-app
kubectl get all -n voting-app
```

- Delete argocd application

```
argocd app delete voting-app
kubectl delete ns voting-app
```

## Deploy application in production with GipOps Principles

- Download repo from git

```
export GH_ORG=gonzaloacosta
git clone https://github.com/$GH_ORG/gitops-voting-app-production.git
cd gitops-voting-app-production
```

- Create ArgoCD manifest project

```
cat << EOF >> project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Production project
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
EOF
```

- Check project resource

```
kubectl get appproject

open http://argocd.$INGRESS_HOST.nip.io/settings/projects
```

### Define an ArgoCD Application like kubernetes objects

- Create Helm templates

```
mkdir -p helm/templates
cat << EOF >> helm/Chart.yaml
apiVersion: v1
description: Production environment
name: voting-app
version: "0.0.1"
EOF
```

- Create Helm manifest for Voting App Resources Application

```
cat << EOF >> helm/templates/voting-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: voting-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    path: helm
    repoURL: https://github.com/gonzaloacosta/voting-app.git
    targetRevision: HEAD
    helm:
      values: |
        result:
          image:
            tag: latest
          replicas: 2
          ingressHost: result.192.168.64.6.nip.io
        vote:
          image:
            tag: latest
          replicas: 2
          ingressHost: vote.192.168.64.6.nip.io
        worker:
          image:
            tag: latest
          replicas: 2
      version: v3
  destination:
    namespace: production
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
EOF
```

### Create Helm Manifest for ArgoCD Production Application

```
cat << EOF >> apps.yaml                                            
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/gonzaloacosta/gitops-voting-app-production.git
    targetRevision: HEAD
    path: helm
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
    - CreateNamespace=true
EOF
```

- Apply configuration

```
kubectl -n argocd apply -f apps.yaml
open http://argocd.$INGRESS_HOST.nip.io
```


### Create Development Environment for Votting App with Gitops Principles

The differente are branch and helm manifest change slightly

- Prepare environment

```
kubectl create ns voting-app-develop
cd voting-app
git checkout -b develop
cat helm/values.yaml | sed -e 'g@image: latest@image: develop@g
docker-compose build .
docker push gonzaloacosta/voting-app-worker:develop
docker push gonzaloacosta/voting-app-result:develop
docker push gonzaloacosta/voting-app-vote:develop
```

- Create argo project and application 

```
cd ../gitops-voting-app-develop
kubectl apply -f project.yaml
kubectl apply -f apps.yaml

open http://vote.develop.$INGRESS_HOST.nip.io
open http://result.develop.$INGRESS_HOST.nip.io
```

### Create Environment without touch K8S for Preview Environments

- Clone repo previews

```
git clone https://github.com/gonzaloacota/gitops-votin-app-previews.git
cd gitops-votin-app-previews
```

- Create Project Previews

```
cat project.yaml
kubectl apply -f project.yaml
```

- Create Images

```
cat helm/values.yaml | sed -e 's@tag: latest@tag: pr-1@g' | helm/values.yaml
cat docker-compose.yml | sed -e 's@:develop@:pr-1@g' | tee docker-compose.yml
docker-compose up -d 
docker push gonzaloacosta/voting-app-worker:pr-1
docker push gonzaloacosta/voting-app-result:pr-1
docker push gonzaloacosta/voting-app-vote:pr-1
```

- Prepare environment preview for PR ID 1

```
export PR_ID=1
export REPO=voting-app
export APP_ID=pr-$REPO-$PR_ID
export IMAGE_TAG=pr-1
export HOSTNAME=$APP_ID.$INGRESS_HOST.nip.io

cat previews.yaml | sed -e "s@_APP_ID_@$APP_ID@g" | sed -e "s@_REPO_@$REPO@g" | sed -e "s@_IMAGE_TAG_@$IMAGE_TAG@g" | sed -e "s@_HOSTNAME_@$HOSTNAME@g" | tee helm/templates/$APP_ID.yaml

cat helm/templates/pr-voting-app-1.yaml

git add .
git commit -m "add pr-1 environment"
git push
```

- Create PR-1 Environment

Sync application without touch kubernetes cluster, only from GitOps principles.

```
argocd app list
argocd sync list previews
kubectl get ns
kubectl get pods -n pr-voting-app-1
open http://argocd.$INGRESS_HOST.nip.io
```

[SyncOptions](https://argoproj.github.io/argo-cd/user-guide/sync-options/)

- Create another PR-2 Environment

Create images
```
cd voting-app/
git checkout develop
git checkout -b pr-2 develop
vim vote/templates/index.html
cat helm/values.yaml | sed -e 's@tag: develop@tag: pr-2@g' | helm/values.yaml
cat docker-compose.yml | sed -e 's@:develop@:pr-2@g' | tee docker-compose.yml 
docker-compose up -d 
docker push gonzaloacosta/voting-app-worker:pr-2
docker push gonzaloacosta/voting-app-result:pr-2
docker push gonzaloacosta/voting-app-vote:pr-2
```

- Create application manifest

```
export PR_ID=2
export REPO=voting-app
export APP_ID=pr-$REPO-$PR_ID
export IMAGE_TAG=pr-2
export HOSTNAME=$APP_ID.$INGRESS_HOST.nip.io

cat previews.yaml | sed -e "s@_APP_ID_@$APP_ID@g" | sed -e "s@_REPO_@$REPO@g" | sed -e "s@_IMAGE_TAG_@$IMAGE_TAG@g" | sed -e "s@_HOSTNAME_@$HOSTNAME@g" | tee helm/templates/$APP_ID.yaml

cat helm/templates/pr-voting-app-2.yaml

git add .
git commit -m "add pr-2 environment"
git push
```

- Check environment

```
argocd app list
argocd app sync previews
kubectl get ns
kubectl get pods -n pr-voting-app-2
```

- Delete an Preview Environment

```
cd gitops-voting-app-previews
rm helm/templates/pr-voting-app-2.yaml

git add .
git commit -m "delete pr-2 environment"
git push

argocd app sync previews

kubectl delete namespace pr-voting-app-2
```

## Conclusions

TODO