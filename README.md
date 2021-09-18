# Deployment Guide

## EKS Cluster
Assume you have eks, aws cli, and kubectl installed and configured

Create Cluster
```
    eksctl create cluster --name shopify-2021-backend --version 1.20 --nodegroup-name ng-default --node-type t2.small --nodes-min 0 --nodes-max 2
```

Select the Cluster
```
aws eks --region us-west-2 update-kubeconfig --name shopify-2021-backend
kubectl config get-clusters
kubectl config set-cluster shopify-2021-backend.us-west-2.eksctl.io
kubectl get pods
```

### ACM: SSL Certificate
Create a SSL certificate for `imagerepo.billzzhang.com`.

## Deploying Applications
### Set Environment Variables
#### Image Repo
```
cp imagerepo/imagerepo-secret.env.sample imagerepo/imagerepo-secret.env
```
Populate .env files `imagerepo-secret.env` from [imagerepo-secret.env.sample](imagerepo/imagerepo-secret.env.sample)

Generate `imagerepo-secret.yaml` based on `imagerepo-secret.yaml.tmpl` and using the values added to `imagerepo-secret.env`
```
sed \
-e "s%TEMPLATE_DB_USERNAME%$(echo $(grep -oP '^DB_USERNAME=\K.*' imagerepo/imagerepo-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_DB_PASSWORD%$(echo $(grep -oP '^DB_PASSWORD=\K.*' imagerepo/imagerepo-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_DB_HOST%$(echo $(grep -oP '^DB_HOST=\K.*' imagerepo/imagerepo-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_DB_NAME%$(echo $(grep -oP '^DB_NAME=\K.*' imagerepo/imagerepo-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_DB_PORT%$(echo $(grep -oP '^DB_PORT=\K.*' imagerepo/imagerepo-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_DB_ADAPTER%$(echo $(grep -oP '^DB_ADAPTER=\K.*' imagerepo/imagerepo-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_SAFE_HOSTS%$(echo $(grep -oP '^SAFE_HOSTS=\K.*' imagerepo/imagerepo-secret.env) | tr -d '\n' |base64 -w0)%g" \
imagerepo/imagerepo-secret.yaml.tmpl > imagerepo/imagerepo-secret.yaml
```

```
cp imagerepo/imagerepo-ingress.env.sample imagerepo/imagerepo-ingress.env
```
Populate .env files `imagerepo-ingress.env` from [imagerepo-ingress.env.sample](imagerepo/imagerepo-ingress.env.sample)

Generate `imagerepo-ingress.yaml` based on `imagerepo-ingress.yaml.tmpl` and using the values added to `imagerepo-ingress.env`
```
sed \
-e "s%TEMPLATE_IMAGE-REPO_SSL_ARN%$(echo $(grep -oP '^IMAGE-REPO_SSL_ARN=\K.*' imagerepo/imagerepo-ingress.env) | tr -d '\n')%g" \
-e "s%TEMPLATE_IMAGE-REPO_HOST%$(echo $(grep -oP '^IMAGE-REPO_HOST=\K.*' imagerepo/imagerepo-ingress.env) | tr -d '\n')%g" \
imagerepo/imagerepo-ingress.yaml.tmpl > imagerepo/imagerepo-ingress.yaml
```


#### Postgres
```
cp postgres/postgres-secret.env.sample postgres/postgres-secret.env
```
Populate .env files `postgres-secret.env` from [postgres-secret.env.sample](ipostgres/postgres-secret.env.sample)

Generate `postgres-secret.yaml` based on `postgres-secret.yaml.tmpl` and using the values added to `postgres-secret.env`
```
sed \
-e "s%TEMPLATE_POSTGRES_USER%$(echo $(grep -oP '^POSTGRES_USER=\K.*' postgres/postgres-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_POSTGRES_PASSWORD%$(echo $(grep -oP '^POSTGRES_PASSWORD=\K.*' postgres/postgres-secret.env) | tr -d '\n' |base64 -w0)%g" \
-e "s%TEMPLATE_POSTGRES_DB%$(echo $(grep -oP '^POSTGRES_DB=\K.*' postgres/postgres-secret.env) | tr -d '\n' |base64 -w0)%g" \
postgres/postgres-secret.yaml.tmpl > postgres/postgres-secret.yaml
```

#### Ingress-nginx controller with TLS
Based on https://github.com/kubernetes/ingress-nginx and more specifically using the examples provided at https://github.com/kubernetes/ingress-nginx/tree/master/deploy/static/provider/aws

Before applying the controller, make sure to add the ACM SSL ARN, and the VPC Network. Lookup for placeholders XXX

```
kubectl apply -f ingress-nginx/deploy.yaml
```

### Apply Applications

#### Image-repo
```
kubectl apply -f imagerepo/imagerepo-secret.yaml

kubectl apply -f imagerepo/imagerepo-deployment.yaml
kubectl apply -f imagerepo/imagerepo-service.yaml
kubectl apply -f imagerepo/imagerepo-ingress.yaml
```

#### Postgres
```
kubectl apply -f storage/pg-storage.yaml
kubectl apply -f postgres/postgres-secret.yaml
kubectl apply -f postgres/postgres-deployment.yaml
kubectl apply -f postgres/postgres-service.yaml
```

### Route 53: DNS Zone
Create the CNAME (only) records as required
```
  imagerepo.billzzhang.com              CNAME       <Ingress imagerepo ADDRESS>
```