# Kubernetes Migration - Quick Reference Card

## 🚀 Quick Deploy (Minikube)

```bash
# 1. Start Minikube
minikube start --memory=4096 --cpus=4

# 2. Build images (each service dir)
mvn clean package -DskipTests
mvn spring-boot:build-image

# 3. Load into Minikube
minikube image load multiplication:0.0.1-SNAPSHOT
minikube image load gamification:0.0.1-SNAPSHOT
minikube image load gateway:0.0.1-SNAPSHOT
minikube image load logs:0.0.1-SNAPSHOT

# 4. Enable ingress
minikube addons enable ingress

# 5. Deploy all
kubectl apply -f k8s/

# 6. Access gateway
kubectl port-forward -n microservices svc/gateway 8000:8000
# http://localhost:8000
```

---

## 🔧 Common Commands

### View Resources
```bash
# All resources
kubectl get all -n microservices

# Just pods
kubectl get pods -n microservices

# Just services
kubectl get svc -n microservices

# ConfigMaps
kubectl get cm -n microservices

# Events
kubectl get events -n microservices --sort-by='.lastTimestamp'
```

### Debugging
```bash
# Pod logs (live)
kubectl logs -n microservices -l app=multiplication -f

# Pod description
kubectl describe pod <pod-name> -n microservices

# Pod shell
kubectl exec -it <pod-name> -n microservices -- bash

# Check environment
kubectl exec <pod-name> -n microservices -- env | grep SPRING
```

### Port Forwarding
```bash
# Gateway
kubectl port-forward -n microservices svc/gateway 8000:8000

# Multiplication
kubectl port-forward -n microservices svc/multiplication 8080:8080

# Zipkin
kubectl port-forward -n microservices svc/zipkin 9411:9411

# Kafka UI
kubectl port-forward -n microservices svc/kafka-ui 8080:8080
```

### Management
```bash
# Scale deployment
kubectl scale deployment multiplication --replicas=3 -n microservices

# Restart deployment
kubectl rollout restart deployment/multiplication -n microservices

# Edit ConfigMap
kubectl edit configmap multiplication-config -n microservices

# Check resource usage
kubectl top pods -n microservices
kubectl top nodes
```

---

## 📋 File Organization

| What | Files |
|------|-------|
| **Namespaces** | `00-namespace.yaml` |
| **Configuration** | `01-configmaps.yaml` |
| **Services** | `02-05` (multiplication, gamification, gateway, logs) |
| **Infrastructure** | `06-07` (kafka, zipkin) |
| **Access** | `08-09` (ingress, kafka-ui) |
| **Bootstrap configs** | `config/*-bootstrap.yml` |

---

## ⚙️ Code Changes Required

### For Each Service (multiplication, gamification, gateway, logs)

**pom.xml:**
```xml
<!-- Remove these -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-consul-config</artifactId>
</dependency>

<!-- Add these -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-kubernetes-client-bootstrap</artifactId>
</dependency>
```

**Configuration:**
- Delete: `bootstrap.properties`
- Add: Copy from `k8s/config/*-bootstrap.yml`
- Remove: All `spring.cloud.consul.*` from `application.properties`

---

## 🐳 Docker Build Methods

### Method 1: Spring Boot Maven Plugin (Recommended)
```bash
cd multiplication
mvn spring-boot:build-image
cd ..
```

### Method 2: Docker CLI
```bash
docker build -t multiplication:0.0.1-SNAPSHOT -f multiplication/Dockerfile multiplication/
```

### Method 3: JIB
```bash
mvn clean compile jib:dockerBuild
```

---

## 🎯 Deployment Steps

### Step 1: Create Namespace
```bash
kubectl apply -f k8s/00-namespace.yaml
```

### Step 2: Configuration
```bash
kubectl apply -f k8s/01-configmaps.yaml
```

### Step 3: Infrastructure (Wait for each to be ready)
```bash
kubectl apply -f k8s/06-kafka-deployment.yaml
kubectl wait --for=condition=Ready pod -l app=kafka -n microservices --timeout=300s

kubectl apply -f k8s/07-zipkin-deployment.yaml
```

### Step 4: Services
```bash
kubectl apply -f k8s/02-multiplication-deployment.yaml
kubectl apply -f k8s/03-gamification-deployment.yaml
kubectl apply -f k8s/04-gateway-deployment.yaml
kubectl apply -f k8s/05-logs-deployment.yaml

# Wait for all to be ready
kubectl wait --for=condition=available --timeout=300s deployment --all -n microservices
```

### Step 5: Access
```bash
kubectl apply -f k8s/08-ingress.yaml
kubectl apply -f k8s/09-kafka-ui-deployment.yaml
```

---

## 🔍 Troubleshooting Quick Fixes

| Problem | Solution |
|---------|----------|
| **Pod not starting** | `kubectl logs <pod> -n microservices` |
| **ImagePullBackOff** | `minikube image load <image>` |
| **Kafka not ready** | Wait longer or check: `kubectl logs -l app=kafka -n microservices` |
| **ConfigMap not loading** | `kubectl delete pod <pod> -n microservices` |
| **Can't connect to service** | `kubectl exec <pod> -n microservices -- nc -zv <service> <port>` |
| **Out of memory** | `minikube start --memory=8192` |

---

## 📊 Service Mapping

| Service | Port | Command | Purpose |
|---------|------|---------|---------|
| **Gateway** | 8000 | `kubectl port-forward svc/gateway 8000:8000` | API entry point |
| **Multiplication** | 8080 | `kubectl port-forward svc/multiplication 8080:8080` | Generate challenges |
| **Gamification** | 8081 | `kubectl port-forward svc/gamification 8081:8081` | Manage scores |
| **Logs** | 8580 | `kubectl port-forward svc/logs 8580:8580` | Centralized logging |
| **Kafka** | 9092 | Internal only | Event streaming |
| **Zipkin** | 9411 | `kubectl port-forward svc/zipkin 9411:9411` | Distributed tracing |
| **Kafka UI** | 8080 | `kubectl port-forward svc/kafka-ui 8080:8080` | Kafka management |

---

## ✅ Verification Checklist

```bash
# 1. Pods Running
kubectl get pods -n microservices
# Expected: All Running status

# 2. Services Healthy
kubectl get svc -n microservices
# Expected: All have ClusterIP/LoadBalancer

# 3. Gateway Response
curl http://localhost:8000/actuator/health
# Expected: 200 OK with UP status

# 4. ConfigMaps Loaded
kubectl get cm -n microservices
# Expected: application, multiplication-config, etc.

# 5. Kafka Ready
kubectl get statefulset kafka -n microservices
# Expected: Ready 1/1

# 6. No Errors
kubectl get events -n microservices | grep -i error
# Expected: No recent error events
```

---

## 🌐 Access Points

### Local Development (Minikube)
```
Gateway:      http://localhost:8000
Multiplication: http://localhost:8080 (via port-forward)
Gamification:  http://localhost:8081 (via port-forward)
Zipkin:        http://localhost:9411 (via port-forward)
Kafka UI:      http://localhost:8080 (via port-forward)
```

### Production (Cloud)
```
Gateway:      http://<INGRESS_IP>:8000
Or via DNS:   http://challenges.example.com
Zipkin:       http://zipkin.<namespace>.svc.cluster.local:9411
Kafka UI:     http://kafka-ui.<namespace>.svc.cluster.local:8080
```

---

## 📝 API Examples

### Get Challenge
```bash
curl http://localhost:8000/challenges
```

### Submit Attempt
```bash
curl -X POST http://localhost:8000/attempts \
  -H "Content-Type: application/json" \
  -d '{
    "factorA": 5,
    "factorB": 3,
    "userAlias": "testuser",
    "resultAttempt": 15
  }'
```

### Get Leaderboard
```bash
curl http://localhost:8000/leaders
```

### Health Check
```bash
curl http://localhost:8000/actuator/health
```

### Metrics
```bash
curl http://localhost:8000/actuator/prometheus
```

---

## 🔄 Update Configuration

### Edit ConfigMap
```bash
# Direct edit
kubectl edit configmap multiplication-config -n microservices

# Or update from file
kubectl apply -f k8s/01-configmaps.yaml
```

### Reload Configuration
```bash
# Restart pod
kubectl delete pod <pod-name> -n microservices

# Or restart deployment
kubectl rollout restart deployment/multiplication -n microservices
```

---

## 🧹 Cleanup

### Stop Services
```bash
# Scale down
kubectl scale deployment multiplication --replicas=0 -n microservices

# Or delete pod
kubectl delete pod <pod-name> -n microservices
```

### Delete Everything
```bash
# Delete namespace (cascades to all resources)
kubectl delete namespace microservices

# Or selectively
kubectl delete deployment --all -n microservices
kubectl delete service --all -n microservices
kubectl delete configmap --all -n microservices
```

### Stop Minikube
```bash
# Pause (keeps state)
minikube pause

# Stop (keeps state)
minikube stop

# Delete (removes everything)
minikube delete
```

---

## 📚 Documentation Map

| Need | Document |
|------|----------|
| Quick local setup | QUICKSTART_MINIKUBE.md |
| Full deployment guide | README.md |
| Verify everything works | DEPLOYMENT_CHECKLIST.md |
| Update Java code | MIGRATION_GUIDE.md |
| Manage configuration | CONFIGMAP_MIGRATION.md |
| Fix problems | TROUBLESHOOTING.md |
| Architecture overview | 00-SUMMARY.md |
| Choose your path | INDEX.md |

---

## 💡 Pro Tips

1. **Watch pods starting:** `watch kubectl get pods -n microservices`
2. **Stream all logs:** `kubectl logs -n microservices -f --all-containers=true`
3. **Kill port forward:** `lsof -i :8000 | grep LISTEN | awk '{print $2}' | xargs kill -9`
4. **Edit in editor:** `kubectl edit configmap multiplication-config -n microservices`
5. **Increase Minikube resources:** `minikube start --memory=8192 --cpus=4 --disk-size=30G`
6. **Access pod without port-forward:** `kubectl run -it --rm debug --image=nicolaka/netcat --restart=Never -- -zv kafka-broker 9092 -n microservices`
7. **See resource limits:** `kubectl get pod <pod-name> -n microservices -o yaml | grep -A10 resources`

---

## ⚡ Most Common Tasks

### Deploy Everything
```bash
kubectl apply -f k8s/
```

### Check Everything is Running
```bash
kubectl get all -n microservices
```

### Debug a Pod
```bash
kubectl logs <pod-name> -n microservices
kubectl describe pod <pod-name> -n microservices
```

### Restart a Service
```bash
kubectl rollout restart deployment/<service> -n microservices
```

### Scale a Service
```bash
kubectl scale deployment/<service> --replicas=3 -n microservices
```

### Access Gateway Locally
```bash
kubectl port-forward svc/gateway 8000:8000 -n microservices
curl http://localhost:8000
```

---

**Print this page and keep it handy! 📌**
