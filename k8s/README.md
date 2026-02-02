# Kubernetes Manifests for Microservices

This directory contains Kubernetes manifests for deploying the Spring Cloud microservices with Spring Cloud Kubernetes instead of Consul.

## File Organization

- **00-namespace.yaml** - Creates the `microservices` namespace
- **01-configmaps.yaml** - All ConfigMaps for application configuration
- **02-multiplication-deployment.yaml** - Multiplication service deployment and service
- **03-gamification-deployment.yaml** - Gamification service deployment and service
- **04-gateway-deployment.yaml** - API Gateway deployment and service
- **05-logs-deployment.yaml** - Logs service deployment and service
- **06-kafka-deployment.yaml** - Kafka StatefulSet and topic initialization job
- **07-zipkin-deployment.yaml** - Zipkin distributed tracing deployment
- **08-ingress.yaml** - Ingress configuration for the gateway
- **09-kafka-ui-deployment.yaml** - Kafka UI for cluster management

## Prerequisites

1. **Kubernetes Cluster** (1.20+)
   - Minikube: `minikube start`
   - Docker Desktop: Enable Kubernetes in settings
   - or any cloud-based K8s cluster

2. **kubectl** configured to access your cluster

3. **NGINX Ingress Controller** (for ingress to work)
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
   ```

4. **Container Images** built and available:
   ```bash
   # Build images locally or push to registry
   docker build -t multiplication:0.0.1-SNAPSHOT -f multiplication/Dockerfile multiplication/
   docker build -t gamification:0.0.1-SNAPSHOT -f gamification/Dockerfile gamification/
   docker build -t gateway:0.0.1-SNAPSHOT -f gateway/Dockerfile gateway/
   docker build -t logs:0.0.1-SNAPSHOT -f logs/Dockerfile logs/
   ```

## Deployment Instructions

### 1. Apply all manifests in order

```bash
# Deploy to default namespace
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-configmaps.yaml
kubectl apply -f k8s/06-kafka-deployment.yaml
kubectl apply -f k8s/07-zipkin-deployment.yaml
kubectl apply -f k8s/02-multiplication-deployment.yaml
kubectl apply -f k8s/03-gamification-deployment.yaml
kubectl apply -f k8s/04-gateway-deployment.yaml
kubectl apply -f k8s/05-logs-deployment.yaml
kubectl apply -f k8s/08-ingress.yaml
kubectl apply -f k8s/09-kafka-ui-deployment.yaml
```

Or apply all at once:
```bash
kubectl apply -f k8s/
```

### 2. Verify deployments

```bash
# Check all resources in microservices namespace
kubectl get all -n microservices

# Check pod status
kubectl get pods -n microservices -w

# Check logs for a specific service
kubectl logs -n microservices -l app=multiplication -f
```

### 3. Wait for services to be ready

```bash
# Check Kafka is ready (required before other services start)
kubectl wait --for=condition=Ready pod -l app=kafka -n microservices --timeout=300s

# Check all pods are running
kubectl get pods -n microservices
```

### 4. Access services

**Gateway (API):**
- Local: `http://localhost:8000`
- Via Ingress: `http://localhost`

**Kafka UI:**
```bash
kubectl port-forward -n microservices svc/kafka-ui 8080:8080
# Access: http://localhost:8080
```

**Zipkin (Tracing):**
```bash
kubectl port-forward -n microservices svc/zipkin 9411:9411
# Access: http://localhost:9411
```

**Multiplication service (direct):**
```bash
kubectl port-forward -n microservices svc/multiplication 8080:8080
# Access: http://localhost:8080
```

**Gamification service (direct):**
```bash
kubectl port-forward -n microservices svc/gamification 8081:8081
# Access: http://localhost:8081
```

## Configuration Management

### How Spring Cloud Kubernetes discovers configuration:

1. **bootstrap.yml** in the application
2. **ConfigMaps** named after the application (e.g., `multiplication-config`)
3. **ConfigMaps** named `application` (shared across all services)
4. **Secrets** for sensitive data (not yet implemented)

### To update configuration:

Edit the ConfigMap and the changes will be picked up (requires restart for most properties, or enable Spring Cloud Kubernetes property reload):

```bash
kubectl edit configmap multiplication-config -n microservices
```

## Important Notes

### Service Discovery
- Services are discovered via Kubernetes DNS: `<service-name>.<namespace>.svc.cluster.local`
- Spring Cloud Kubernetes automatically handles this
- No need for Consul

### Environment Variables
The following are set via deployment env variables:
- `SPRING_CLOUD_KAFKA_HOST=kafka-broker`
- `SPRING_CLOUD_ZIPKIN_HOST=zipkin`
- `SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka-broker:9092`

### Health Checks
- Liveness probe: `/actuator/health/liveness`
- Readiness probe: `/actuator/health/readiness`
- Startup probe: `/actuator/health`

### Storage
- Kafka uses a PersistentVolumeClaim (10Gi by default)
- Multiplication and Gamification use in-memory H2 or file-based storage
- For production, implement proper persistent storage

## Cleanup

To remove all resources:

```bash
kubectl delete namespace microservices
```

## Troubleshooting

### Services not starting
```bash
# Check pod events and logs
kubectl describe pod <pod-name> -n microservices
kubectl logs <pod-name> -n microservices
```

### ConfigMap not being loaded
```bash
# Verify ConfigMap exists
kubectl get configmap -n microservices
kubectl describe configmap <config-name> -n microservices
```

### Kafka connectivity issues
```bash
# Test Kafka connectivity from a pod
kubectl run -it --rm debug --image=nicolaka/netcat --restart=Never -- -zv kafka-broker 9092 -n microservices
```

### Port forwarding for local access
```bash
# Gateway
kubectl port-forward -n microservices svc/gateway 8000:8000

# Multiplication
kubectl port-forward -n microservices svc/multiplication 8080:8080
```

## Next Steps

1. Update service bootstrap.yml files to use Spring Cloud Kubernetes
2. Remove spring-cloud-consul dependencies from pom.xml
3. Add spring-cloud-starter-kubernetes-client dependencies
4. Test locally with Minikube
5. Create docker images from each service
6. Deploy to your Kubernetes cluster
