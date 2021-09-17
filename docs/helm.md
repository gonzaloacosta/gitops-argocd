# Helm

## Environments

- Development:
    - Ingress: <gitlab_user>.<domain>
    - HPA: false
    - Replicas: 1
    - DB Replicas: 1
    - DB Persistance: false
- Staging:
    - Ingress: staging.<domain>
    - HPA: true
    - Replicas: 2
    - DB Replicas: 1
    - DB Persistance: false
- Production:
    - Ingress: <app_name>.<domain>
    - HPA: true
    - Replicas: 4
    - DB Replicas: 1
    - DB Persistance: true

## Production

```
export INGRESS_HOST=$(minikube ip)
helm show readme bitnami/redis
helm repo list
helm search repo redis
helm show values bitnami/redis
helm show readme bitnami/postgresql
helm repo list
helm search repo postgresql
helm show values bitnami/postgresql
helm dependency list voting-app
helm dependency update voting-app
helm dependency list voting-app
helm --namespace production upgrade --install voting-app voting-app --wait --timeout 10m

kubectl get ingress voting-app-voting-app
kubectl get ingress,hpa,pods -n production
curl -H "Host: voting-app.example.com" http://$INGRESS_HOST
```

## Development

```
export GH_USER=gonzaloacosta
kubectl create namespace $GH_USER
cat dev/values.yaml
export ADDR=$GH_USER.voting-app.example.com
helm --namespace $GH_USER upgrade --install --value dev/values.yaml --set ingress.host=$ADDR voting-app voting-app --wait --timeout 10m
kubectl -n $GH_USER get ingress,pod,hpa,pvc
curl -H "Host: gonzaloacosta.voting-app.example.com " http://$INGRESS_HOST
```

## Staging 

```
kubectl create ns staging
cat staging/values.yaml
helm --namespace staging upgrade --install --values staging/values.yaml voting-app voting-app --wait --timeout 10m
kubectl -n staging get ingress,pod,hpa,pvc
curl -H "Host: staging.voting-app.example.com" http://$INGRESS_HOST
```

## Packaging

```
cat voting-app/Chart.yaml \
  | sed -e "s@version: 0.0.1@version: 0.0.2@g" \
  | sed -e "s@appVersion: 0.0.1@appVersion: 0.0.2@g" \
  | tee voting-app/Chart.yaml

cat voting-app/values.yaml \ 
  | sed -e "s@tag: 0.0.1@tag: 0.0.2@g" \ 
  | tee voting-app/values.yaml

helm lint voting-app

helm package voting-app
```

## Upgrade production

```
helm --namespace production upgrade --install voting-app voting-app-0.0.2.tgz --wait --timeout 10m
curl -H "Host: voting-app.example.com" http://$INGRESS_HOST
helm --namespace production get all voting-app
```

## Rolling Back Releasese

```
helm --namespace production history voting-app
helm --namespace production rollback voting-app
helm --namespace production history voting-app
```
