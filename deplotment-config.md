# Kubernetes Deployment - Complete Setup Guide

## Prerequisites

```bash
# 1. Povećajte Minikube memoriju (OBAVEZNO!)
minikube stop
minikube delete
minikube start --memory=15360 --cpus=4

# 2. Enable Ingress
minikube addons enable ingress

# 3. Kreirajte Docker Registry Secret
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_PAT \
  --docker-email=YOUR_EMAIL
```

## Deployment Order

### 1. Infrastructure Services (PostgreSQL databases)

```bash
# PostgreSQL za Identity
helm install postgres postgres-identity-chart

# PostgreSQL za ostale servise
helm install accommodation-postgres accommodation-postgres-chart
helm install booking-postgres booking-postgres-chart
helm install notification-postgres notification-postgres-chart
helm install review-postgres review-postgres-chart

# Sačekajte da budu ready
kubectl wait --for=condition=ready pod -l app=postgres --timeout=120s
kubectl wait --for=condition=ready pod -l app=accommodation-postgres --timeout=120s
kubectl wait --for=condition=ready pod -l app=booking-postgres --timeout=120s
kubectl wait --for=condition=ready pod -l app=notification-postgres --timeout=120s
kubectl wait --for=condition=ready pod -l app=review-postgres --timeout=120s
```

### 2. Keycloak (Authentication)

```bash
helm install keycloak keycloak-chart
kubectl wait --for=condition=ready pod -l app=keycloak --timeout=180s
```

### 3. Kafka & Zookeeper

```bash
helm install kafka kafka-chart
kubectl wait --for=condition=ready pod -l app=zookeeper --timeout=120s
kubectl wait --for=condition=ready pod -l app=kafka --timeout=120s
```

### 4. Elasticsearch & Kibana

```bash
helm install elasticsearch elasticsearch-chart
kubectl wait --for=condition=ready pod -l app=elasticsearch --timeout=180s

# Kibana (optional)
helm install kibana kibana-chart
```

### 5. Backend Services

```bash
# Identity Service
helm install identity-service backend-chart

# Accommodation Service
helm install accommodation-service accommodation-chart

# Booking Service
helm install booking-service booking-chart

# Notification Service
helm install notification-service notification-chart

# Review Service
helm install review-service review-chart

# Search Service
helm install search-service search-chart

# Sačekajte da budu ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=identity-service --timeout=180s
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=accommodation-service --timeout=180s
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=booking-service --timeout=180s
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=notification-service --timeout=180s
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=review-service --timeout=180s
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=search-service --timeout=180s
```

### 6. Frontend

```bash
helm install frontend-app frontend-chart
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=frontend-app --timeout=120s
```

## Verification

```bash
# Check all pods
kubectl get pods

# Check services
kubectl get svc

# Check ingress
kubectl get ingress

# Get Minikube tunnel (za pristup ingress-u)
minikube tunnel
```

## Accessing Services

```bash
# Get Minikube IP
minikube ip

# Add to /etc/hosts (ako koristite hostnames)
echo "$(minikube ip) your-app.local" | sudo tee -a /etc/hosts

# Ili pristupite direktno preko Minikube IP-a
```

## Troubleshooting

```bash
# Logovi specifičnog servisa
kubectl logs -f deployment/SERVICE_NAME

# Describe pod za detalje
kubectl describe pod POD_NAME

# Restart deployment
kubectl rollout restart deployment/SERVICE_NAME

# Delete i reinstall
helm uninstall SERVICE_NAME
helm install SERVICE_NAME CHART_NAME
```

## Update Image Tags

```bash
# Update single service
helm upgrade SERVICE_NAME CHART_NAME --set image.tag=new-sha

# Example
helm upgrade identity-service backend-chart --set image.tag=sha-abc123
```

## Cleanup All

```bash
# Delete all Helm releases
helm uninstall frontend-app
helm uninstall search-service
helm uninstall review-service
helm uninstall notification-service
helm uninstall booking-service
helm uninstall accommodation-service
helm uninstall identity-service
helm uninstall kibana
helm uninstall elasticsearch
helm uninstall kafka
helm uninstall keycloak
helm uninstall review-postgres
helm uninstall notification-postgres
helm uninstall booking-postgres
helm uninstall accommodation-postgres
helm uninstall postgres

# Delete PVCs
kubectl delete pvc --all

# Delete secrets
kubectl delete secret ghcr-secret
```

## Common Issues

**OOMKilled**: Povećajte memory u chart values.yaml i upgrade
**ImagePullBackOff**: Proverite ghcr-secret i image tag
**CrashLoopBackOff**: Proverite logove - verovatno probe ili dependency problem

**KRITIČNO**: Minikube MORA imati minimum 8GB RAM-a, inače će Elasticsearch i Kafka stalno padati zbog OOM.