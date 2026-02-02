# Quick Start Guide: Local Kubernetes Development with Minikube

Get the microservices running locally on Kubernetes in minutes.

## Prerequisites (5 minutes)

### 1. Install Minikube
```bash
# macOS
brew install minikube

# Windows (using Chocolatey)
choco install minikube

# Linux
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### 2. Install kubectl
```bash
# macOS
brew install kubectl

# Windows (using Chocolatey)
choco install kubernetes-cli

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### 3. Install Docker (if not already installed)
- Download from https://www.docker.com/products/docker-desktop

## Start Minikube (1 minute)

```bash
# Start Minikube with enough resources
minikube start --memory=4096 --cpus=4 --disk-size=20G

# Verify
kubectl cluster-info
kubectl get nodes
```

Expected output:
```
NAME       STATUS   ROLES           AGE     VERSION
minikube   Ready    control-plane   XXs     vX.XX.X
```

## Build Docker Images (3-5 minutes)

```bash
# Build each service
cd multiplication && mvn clean package -DskipTests && cd ..
cd gamification && mvn clean package -DskipTests && cd ..
cd gateway && mvn clean package -DskipTests && cd ..
cd logs && mvn clean package -DskipTests && cd ..
```

## Load Images into Minikube (2 minutes)

```bash
# Build with Spring Boot
cd multiplication && mvn spring-boot:build-image && cd ..
cd gamification && mvn spring-boot:build-image && cd ..
cd gateway && mvn spring-boot:build-image && cd ..
cd logs && mvn spring-boot:build-image && cd ..

# Or build with Docker and load
docker build -t multiplication:0.0.1-SNAPSHOT multiplication/
docker build -t gamification:0.0.1-SNAPSHOT gamification/
docker build -t gateway:0.0.1-SNAPSHOT gateway/
docker build -t logs:0.0.1-SNAPSHOT logs/

minikube image load multiplication:0.0.1-SNAPSHOT
minikube image load gamification:0.0.1-SNAPSHOT
minikube image load gateway:0.0.1-SNAPSHOT
minikube image load logs:0.0.1-SNAPSHOT

# Verify images loaded
minikube image ls | grep -E "(multiplication|gamification|gateway|logs)"
```

## Install NGINX Ingress Controller (1 minute)

```bash
# Enable ingress addon for Minikube
minikube addons enable ingress

# Verify
kubectl get pods -n ingress-nginx
```

## Deploy to Kubernetes (2 minutes)

```bash
# Apply all manifests
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

## Wait for Services to Be Ready (2-5 minutes)

```bash
# Watch pods starting
kubectl get pods -n microservices -w

# Wait for all to be Running
kubectl wait --for=condition=Ready pod -l app=kafka -n microservices --timeout=300s
kubectl wait --for=condition=available --timeout=300s deployment --all -n microservices
```

## Access Services

### Gateway (Main entry point)
```bash
# Get Minikube IP
minikube ip  # e.g., 192.168.49.2

# Test gateway
curl http://$(minikube ip):8000/actuator/health
curl http://$(minikube ip):8000/users

# Or use port forwarding
kubectl port-forward -n microservices svc/gateway 8000:8000
# Then access: http://localhost:8000
```

### Multiplication Service
```bash
kubectl port-forward -n microservices svc/multiplication 8080:8080
# Access: http://localhost:8080/actuator/health
```

### Gamification Service
```bash
kubectl port-forward -n microservices svc/gamification 8081:8081
# Access: http://localhost:8081/actuator/health
```

### Kafka UI
```bash
kubectl port-forward -n microservices svc/kafka-ui 8080:8080
# Access: http://localhost:8080
```

### Zipkin (Distributed Tracing)
```bash
kubectl port-forward -n microservices svc/zipkin 9411:9411
# Access: http://localhost:9411
```

## Test the Application

### 1. Get multiplication challenge
```bash
curl -X GET http://localhost:8000/challenges \
  -H "Content-Type: application/json"
```

### 2. Submit an attempt
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

### 3. Check leaderboard
```bash
curl -X GET http://localhost:8000/leaders
```

## Common Commands

### View logs
```bash
# Logs from multiplication service
kubectl logs -n microservices -l app=multiplication -f

# Logs from all services
kubectl logs -n microservices -f --all-containers=true

# Previous logs if pod crashed
kubectl logs -n microservices <pod-name> --previous
```

### Describe pod for debugging
```bash
kubectl describe pod <pod-name> -n microservices
```

### Execute command in pod
```bash
kubectl exec -it <pod-name> -n microservices -- /bin/sh
```

### Check pod resource usage
```bash
kubectl top pods -n microservices
```

### Update configuration
```bash
# Edit ConfigMap directly
kubectl edit configmap multiplication-config -n microservices

# Pod restarts to pick up changes (if reload enabled)
# Or manually restart deployment
kubectl rollout restart deployment/multiplication -n microservices
```

### View all resources
```bash
kubectl get all -n microservices
```

## Cleaning Up

### Stop Minikube (keeps configuration)
```bash
minikube stop
```

### Delete everything
```bash
kubectl delete namespace microservices
```

### Completely reset Minikube
```bash
minikube delete
```

## Troubleshooting

### Pods not starting
```bash
# Check pod status
kubectl describe pod <pod-name> -n microservices

# Check for image pull errors
kubectl get events -n microservices

# Verify images are loaded
minikube image ls
```

### OutOfMemory errors
```bash
# Increase Minikube memory
minikube start --memory=8192
```

### Kafka connection issues
```bash
# Check if Kafka is running
kubectl get pods -n microservices -l app=kafka

# Check Kafka logs
kubectl logs -n microservices -l app=kafka

# Test connectivity from pod
kubectl exec -it <pod-name> -n microservices -- \
  nc -zv kafka-broker 9092
```

### ConfigMap not loading
```bash
# Verify ConfigMap exists
kubectl get configmap -n microservices
kubectl describe configmap multiplication-config -n microservices

# Restart pod
kubectl delete pod <pod-name> -n microservices
```

### Port forward not working
```bash
# Kill existing port forwards
lsof -i :8000
kill -9 <PID>

# Or try again
kubectl port-forward -n microservices svc/gateway 8000:8000
```

## Next Steps

1. **Modify configuration**: Edit ConfigMaps in `k8s/01-configmaps.yaml`
2. **Scale services**: `kubectl scale deployment multiplication --replicas=3 -n microservices`
3. **Monitor metrics**: Check `http://localhost:8000/actuator/prometheus`
4. **View traces**: Open Zipkin at `http://localhost:9411`
5. **Explore Kafka**: Use Kafka UI at `http://localhost:8080`

## Tips

- Use `watch` command to monitor changes: `watch kubectl get pods -n microservices`
- Access pod shell for debugging: `kubectl exec -it <pod-name> -n microservices -- bash`
- Use `kubectl logs -f` to stream logs in real-time
- Port forward multiple services in separate terminals for convenience
- Use `kubectl port-forward -n microservices --address 0.0.0.0 svc/gateway 8000:8000` to access from other machines

## Documentation References

- Full documentation: `README.md`
- Deployment checklist: `DEPLOYMENT_CHECKLIST.md`
- ConfigMap migration: `CONFIGMAP_MIGRATION.md`
- Migration guide: `MIGRATION_GUIDE.md`
