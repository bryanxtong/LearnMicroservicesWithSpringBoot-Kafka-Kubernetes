# Kubernetes Deployment Checklist

## Pre-Deployment Requirements

- [ ] Kubernetes cluster installed (1.20+)
  - [ ] Minikube: `minikube start --memory=4096 --cpus=4`
  - [ ] Docker Desktop: Kubernetes enabled
  - [ ] Cloud provider (AWS EKS, GCP GKE, Azure AKS): cluster created

- [ ] `kubectl` installed and configured
  ```bash
  kubectl cluster-info
  kubectl get nodes
  ```

- [ ] NGINX Ingress Controller installed
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
  ```

- [ ] Container runtime accessible (Docker daemon running)

## Service Preparation

### Code Changes

- [ ] **multiplication service**
  - [ ] Replace `multiplication/src/main/resources/bootstrap.properties` with `k8s/config/multiplication-bootstrap.yml`
  - [ ] Update `pom.xml`: Remove Consul, add Kubernetes dependencies
  - [ ] Remove `spring.cloud.consul.*` from `application.properties`
  - [ ] Clean and build: `mvn clean package -DskipTests`

- [ ] **gamification service**
  - [ ] Replace `gamification/src/main/resources/bootstrap.properties` with `k8s/config/gamification-bootstrap.yml`
  - [ ] Update `pom.xml`: Remove Consul, add Kubernetes dependencies
  - [ ] Remove `spring.cloud.consul.*` from `application.properties`
  - [ ] Clean and build: `mvn clean package -DskipTests`

- [ ] **gateway service**
  - [ ] Replace `gateway/src/main/resources/bootstrap.yml` with `k8s/config/gateway-bootstrap.yml`
  - [ ] Update `pom.xml`: Remove Consul, add Kubernetes dependencies
  - [ ] Clean and build: `mvn clean package -DskipTests`

- [ ] **logs service**
  - [ ] Replace `logs/src/main/resources/bootstrap.properties` with `k8s/config/logs-bootstrap.yml` (if exists)
  - [ ] Update `pom.xml`: Remove Consul, add Kubernetes dependencies
  - [ ] Remove `spring.config.import=optional:consul:` from `application.properties`
  - [ ] Clean and build: `mvn clean package -DskipTests`

### Docker Image Building

Choose one method:

**Option 1: Using Spring Boot Maven Plugin**
```bash
cd multiplication && mvn spring-boot:build-image && cd ..
cd gamification && mvn spring-boot:build-image && cd ..
cd gateway && mvn spring-boot:build-image && cd ..
cd logs && mvn spring-boot:build-image && cd ..
```

**Option 2: Using Docker directly**
```bash
# Build each image
docker build -t multiplication:0.0.1-SNAPSHOT -f multiplication/Dockerfile multiplication/
docker build -t gamification:0.0.1-SNAPSHOT -f gamification/Dockerfile gamification/
docker build -t gateway:0.0.1-SNAPSHOT -f gateway/Dockerfile gateway/
docker build -t logs:0.0.1-SNAPSHOT -f logs/Dockerfile logs/
```

**Option 3: Using JIB**
```bash
mvn clean compile jib:dockerBuild
```

- [ ] Verify images built successfully
  ```bash
  docker images | grep -E "(multiplication|gamification|gateway|logs)"
  ```

- [ ] For Minikube: Load images into Minikube
  ```bash
  minikube image load multiplication:0.0.1-SNAPSHOT
  minikube image load gamification:0.0.1-SNAPSHOT
  minikube image load gateway:0.0.1-SNAPSHOT
  minikube image load logs:0.0.1-SNAPSHOT
  ```

## Kubernetes Deployment

### Phase 1: Infrastructure Setup

```bash
# Create namespace
kubectl apply -f k8s/00-namespace.yaml

# Verify namespace created
kubectl get ns microservices
```

- [ ] Namespace `microservices` created successfully

### Phase 2: Configuration

```bash
# Apply all ConfigMaps
kubectl apply -f k8s/01-configmaps.yaml

# Verify ConfigMaps created
kubectl get configmaps -n microservices
```

- [ ] ConfigMap `application` created
- [ ] ConfigMap `multiplication-config` created
- [ ] ConfigMap `gamification-config` created
- [ ] ConfigMap `gateway-config` created
- [ ] ConfigMap `logs-config` created

### Phase 3: Stateful Services (Kafka, Zipkin)

```bash
# Deploy Kafka first (required by all services)
kubectl apply -f k8s/06-kafka-deployment.yaml

# Wait for Kafka to be ready
kubectl wait --for=condition=Ready pod -l app=kafka -n microservices --timeout=300s

# Check Kafka status
kubectl get statefulset kafka -n microservices
kubectl get pvc -n microservices
```

- [ ] Kafka StatefulSet created
- [ ] Kafka pod running
- [ ] Kafka topics initialization job completed
- [ ] PersistentVolumeClaim created for Kafka storage

```bash
# Deploy Zipkin (for distributed tracing)
kubectl apply -f k8s/07-zipkin-deployment.yaml

# Verify Zipkin is running
kubectl get deployment zipkin -n microservices
```

- [ ] Zipkin deployment created and running

### Phase 4: Microservices

```bash
# Deploy all microservices
kubectl apply -f k8s/02-multiplication-deployment.yaml
kubectl apply -f k8s/03-gamification-deployment.yaml
kubectl apply -f k8s/04-gateway-deployment.yaml
kubectl apply -f k8s/05-logs-deployment.yaml

# Watch deployment progress
kubectl get deployments -n microservices -w

# Check all pods
kubectl get pods -n microservices
```

- [ ] Multiplication deployment created: 2 replicas
- [ ] Gamification deployment created: 2 replicas
- [ ] Gateway deployment created: 2 replicas
- [ ] Logs deployment created: 1 replica

### Phase 5: Verify All Services Running

```bash
# Wait for all deployments to be ready
kubectl wait --for=condition=available --timeout=300s deployment/multiplication -n microservices
kubectl wait --for=condition=available --timeout=300s deployment/gamification -n microservices
kubectl wait --for=condition=available --timeout=300s deployment/gateway -n microservices
kubectl wait --for=condition=available --timeout=300s deployment/logs -n microservices

# Check overall status
kubectl get all -n microservices
```

- [ ] All pods are in Running state
- [ ] All pods passed readiness probes
- [ ] No pods in CrashLoopBackOff state

### Phase 6: Ingress and External Access

```bash
# Apply ingress configuration
kubectl apply -f k8s/08-ingress.yaml

# Deploy Kafka UI
kubectl apply -f k8s/09-kafka-ui-deployment.yaml

# Check ingress status
kubectl get ingress -n microservices
```

- [ ] Ingress created and has external IP
- [ ] Kafka UI deployment created

## Post-Deployment Verification

### Check Service Health

```bash
# Check endpoint health
kubectl get endpoints -n microservices

# Check services
kubectl get services -n microservices

# Port forward to gateway
kubectl port-forward -n microservices svc/gateway 8000:8000 &

# Test gateway health
curl http://localhost:8000/actuator/health
```

- [ ] Gateway responds to health check
- [ ] All services are discoverable via Kubernetes DNS

### Check Logs

```bash
# Check multiplication service logs
kubectl logs -n microservices -l app=multiplication --tail=50

# Check gamification service logs
kubectl logs -n microservices -l app=gamification --tail=50

# Check gateway logs
kubectl logs -n microservices -l app=gateway --tail=50

# Check logs service
kubectl logs -n microservices -l app=logs --tail=50

# Check for errors
kubectl logs -n microservices -l app=multiplication --tail=100 | grep -i error
```

- [ ] No startup errors in logs
- [ ] Services connected to Kafka
- [ ] Services connected to Zipkin
- [ ] No connection timeouts

### Test Connectivity

```bash
# Test multiplication service
kubectl port-forward -n microservices svc/multiplication 8080:8080 &
curl http://localhost:8080/actuator/health
curl http://localhost:8080/users

# Test gamification service
kubectl port-forward -n microservices svc/gamification 8081:8081 &
curl http://localhost:8081/actuator/health
curl http://localhost:8081/leaders

# Test gateway routing
curl http://localhost:8000/users
curl http://localhost:8000/leaders
```

- [ ] Multiplication service responds
- [ ] Gamification service responds
- [ ] Gateway routing works
- [ ] CORS enabled for frontend

### Monitor Metrics

```bash
# Check resource usage
kubectl top nodes
kubectl top pods -n microservices

# Check Prometheus metrics (if enabled)
curl http://localhost:8000/actuator/prometheus
```

- [ ] CPU usage reasonable
- [ ] Memory usage within limits
- [ ] No OOM kills

## Optional: Access UIs

### Kafka UI

```bash
kubectl port-forward -n microservices svc/kafka-ui 8080:8080
# Access: http://localhost:8080
```

- [ ] Kafka UI accessible
- [ ] Topics visible: `attempts.topic`, `logs`
- [ ] Broker healthy

### Zipkin Tracing

```bash
kubectl port-forward -n microservices svc/zipkin 9411:9411
# Access: http://localhost:9411
```

- [ ] Zipkin accessible
- [ ] Traces are being received
- [ ] Traces show correct service names

### Direct Service Access

```bash
# Create a tunnel to multiplication
kubectl port-forward -n microservices svc/multiplication 8080:8080

# Create a tunnel to gamification
kubectl port-forward -n microservices svc/gamification 8081:8081

# Create a tunnel to gateway
kubectl port-forward -n microservices svc/gateway 8000:8000

# Access H2 console (if enabled)
curl http://localhost:8080/h2-console
```

## Troubleshooting

### Pods not starting

```bash
# Check pod events
kubectl describe pod <pod-name> -n microservices

# Check image pull errors
kubectl get events -n microservices --sort-by='.lastTimestamp'

# Check logs for startup errors
kubectl logs <pod-name> -n microservices
```

- [ ] Images are available
- [ ] ConfigMaps are mounted correctly
- [ ] Environment variables are set

### Connectivity issues

```bash
# Test DNS resolution from a pod
kubectl exec -it <pod-name> -n microservices -- nslookup kafka-broker

# Test connectivity to Kafka
kubectl exec -it <pod-name> -n microservices -- nc -zv kafka-broker 9092

# Check service endpoints
kubectl get endpoints kafka-broker -n microservices
```

- [ ] DNS resolution works
- [ ] Services are accessible from pods

### ConfigMap not loading

```bash
# Check if ConfigMap exists
kubectl get configmap -n microservices
kubectl describe configmap multiplication-config -n microservices

# Check pod environment variables
kubectl exec -it <pod-name> -n microservices -- env | grep SPRING

# Restart pod to reload config
kubectl delete pod <pod-name> -n microservices
```

- [ ] ConfigMap exists in correct namespace
- [ ] ConfigMap has correct keys
- [ ] Pod can mount ConfigMap

## Cleanup (If needed)

```bash
# Delete all resources in namespace
kubectl delete namespace microservices

# Or delete specific resources
kubectl delete deployment -n microservices --all
kubectl delete services -n microservices --all
kubectl delete configmap -n microservices --all
kubectl delete statefulset -n microservices --all
kubectl delete pvc -n microservices --all
```

- [ ] All resources deleted successfully
- [ ] No orphaned volumes or resources

## Final Verification Checklist

- [ ] All pods Running
- [ ] All deployments Ready
- [ ] All services have endpoints
- [ ] ConfigMaps mounted in pods
- [ ] Kafka topics created
- [ ] Gateway routes working
- [ ] Health checks passing
- [ ] Logs showing normal operation
- [ ] Metrics available
- [ ] Tracing working in Zipkin
- [ ] No persistent errors in logs

## Success Criteria

✅ **Deployment is successful when:**
1. All microservices are running
2. Gateway responds to requests
3. Services can communicate via Kubernetes DNS
4. Kafka topics exist and are accessible
5. Zipkin receives traces
6. No CrashLoopBackOff pods
7. Configuration loaded from ConfigMaps
8. Frontend can connect through gateway (if deployed)

## Notes

- Images with `IfNotPresent` pull policy: Requires pre-built images locally or in registry
- For production: Change to pull policy `Always` and use a container registry
- For Minikube: Use `imagePullPolicy: Never` and load images with `minikube image load`
- For cloud: Push images to container registry (Docker Hub, ECR, GCR, ACR)
