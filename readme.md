# CLUSTER KUBERNETES AKS
# **********************************************************
# nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -f lib/nginx-values.yaml

# namespaces
kubectl create namespace qa
kubectl create namespace prod

# Dashboard
kubectl apply -f dashboard/dashboard-adminuser.yaml
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"

# Access dashboard
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

# test template
helm install api-comments ./deploy-app/api-chart -n qa -f ./deploy-app/values-qa.yaml --dry-run --debug
helm install api-comments ./deploy-app/api-chart -n prod -f ./deploy-app/values-prod.yaml --dry-run --debug

# Apply deploy - QA
helm install api-comments ./deploy-app/api-chart -n qa -f ./deploy-app/values-qa.yaml
helm upgrade api-comments ./deploy-app/api-chart -n qa -f ./deploy-app/values-qa.yaml
kubectl get all -n qa

# Apply deploy - PROD
helm install api-comments ./deploy-app/api-chart -n prod -f ./deploy-app/values-prod.yaml
helm upgrade api-comments ./deploy-app/api-chart -n prod -f ./deploy-app/values-prod.yaml
kubectl get all -n prod

## CERTIFICATE SSL
# Create a dedicated Kubernetes namespace for cert-manager
kubectl create namespace cert-manager
# Add official cert-manager repository to helm CLI
helm repo add jetstack https://charts.jetstack.io
# Update Helm repository cache (think of apt update)
helm repo update
# Install cert-manager on Kubernetes
## cert-manager relies on several Custom Resource Definitions (CRDs)
## Helm can install CRDs in Kubernetes (version >= 1.15) starting with Helm version 3.2
helm install certmgr jetstack/cert-manager \
    --set installCRDs=true \
    --version v1.0.4 \
    --namespace cert-manager

kubectl apply -f lib/ssl/issuer.yaml
kubectl apply -f lib/ssl/certificates.yaml

## Prometheus and Grafana installation
kubectl create namespace monitoramento
kubectl apply -f monitoramento/prometheus/cluster-role.yaml
# Check installed
kubectl get clusterroles -n monitoramento
# StorageClass
kubectl apply -f monitoramento/prometheus/storage-class.yaml
kubectl get storageclass
# PVC
kubectl apply -f monitoramento/prometheus/prometheus-pvc.yaml
# Check pvc
kubectl get pvc -n monitoramento
kubectl get pv -n monitoramento
# ConfigMap
kubectl create configmap prometheus-configmap --from-file=monitoramento/prometheus/config/prometheus.yaml -n monitoramento
kubectl get configmap -n monitoramento
kubectl describe configmap prometheus-configmap -n monitoramento
# Deploy
kubectl apply -f monitoramento/prometheus/deployment.yaml
kubectl get deployment -n monitoramento
kubectl describe deployment prometheus-deploy -n monitoramento
# Service Prometheus
kubectl apply -f monitoramento/prometheus/service.yaml

# Install Grafana
kubectl apply -f monitoramento/monitoramento/grafana/

# Install datasource
http://prometheus-service:9090

# Import dashboards
10000 e 8588