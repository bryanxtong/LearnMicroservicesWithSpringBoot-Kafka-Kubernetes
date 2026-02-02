# Kubernetes Troubleshooting Guide

Common issues and solutions when deploying to Kubernetes.

## Pod Status Issues

### CrashLoopBackOff

**Symptom:** Pod keeps restarting

**Check logs:**
```bash
kubectl logs <pod-name> -n microservices
kubectl logs <pod-name> -n microservices --previous
kubectl describe pod <pod-name> -n microservices
```

**Common causes:**

1. **Kafka not ready**
   - Solution: Wait for Kafka to be running
   ```bash
   kubectl wait --for=condition=Ready pod -l app=kafka -n microservices --timeout=300s
   ```

2. **ConfigMap missing**
   - Solution: Apply ConfigMaps
   ```bash
   kubectl apply -f k8s/01-configmaps.yaml
   kubectl get configmap -n microservices
   ```

3. **Wrong Spring Cloud Kubernetes version**
   - Check: `mvn dependency:tree | grep kubernetes`
   - Ensure: Spring Cloud version matches Spring Boot version

4. **Missing environment variables**
   - Check: `kubectl exec -it <pod-name> -n microservices -- env | grep SPRING`
   - Ensure: All required env vars are set in deployment

### Pending

**Symptom:** Pod stuck in Pending state

**Check:**
```bash
kubectl describe pod <pod-name> -n microservices
```

**Common causes:**

1. **Not enough resources**
   - Solution: Increase cluster resources or reduce resource requests
   ```bash
   kubectl describe node
   kubectl top nodes
   ```

2. **PVC (Persistent Volume Claim) pending**
   - Solution: Check if storage is available
   ```bash
   kubectl get pvc -n microservices
   kubectl describe pvc <pvc-name> -n microservices
   ```

3. **Node not ready**
   - Solution: Check node status
   ```bash
   kubectl get nodes
   kubectl describe node <node-name>
   ```

### ImagePullBackOff

**Symptom:** Pod can't pull container image

**Check:**
```bash
kubectl describe pod <pod-name> -n microservices
```

**Common causes:**

1. **Image doesn't exist**
   - Solution: Build and load image
   ```bash
   # For Minikube
   minikube image load multiplication:0.0.1-SNAPSHOT
   minikube image ls
   ```

2. **Wrong image name or tag**
   - Check: Deployment YAML has correct image name
   - Solution: Update deployment with correct image

3. **Private registry credentials needed**
   - Solution: Create ImagePullSecret
   ```bash
   kubectl create secret docker-registry regcred \
     --docker-server=<registry> \
     --docker-username=<user> \
     --docker-password=<pass> \
     -n microservices
   ```

## Connectivity Issues

### Services can't connect to Kafka

**Symptom:** Application logs show Kafka connection errors

**Check:**
```bash
# 1. Verify Kafka is running
kubectl get pods -n microservices -l app=kafka

# 2. Check Kafka service endpoints
kubectl get endpoints kafka-broker -n microservices

# 3. Test connectivity from pod
kubectl exec -it <pod-name> -n microservices -- nc -zv kafka-broker 9092

# 4. Check Kafka logs
kubectl logs -n microservices -l app=kafka
```

**Solution:**
```bash
# Restart Kafka
kubectl delete statefulset kafka -n microservices
kubectl apply -f k8s/06-kafka-deployment.yaml

# Wait for it to be ready
kubectl wait --for=condition=Ready pod -l app=kafka -n microservices --timeout=300s

# Restart services
kubectl rollout restart deployment/multiplication -n microservices
kubectl rollout restart deployment/gamification -n microservices
```

### Services can't connect to each other

**Symptom:** Gateway can't route to multiplication/gamification

**Check:**
```bash
# 1. Check service exists
kubectl get svc -n microservices

# 2. Check endpoints
kubectl get endpoints -n microservices

# 3. Test DNS resolution from pod
kubectl exec -it <pod-name> -n microservices -- nslookup multiplication

# 4. Test connectivity
kubectl exec -it <pod-name> -n microservices -- nc -zv multiplication 8080
```

**Solution:**
```bash
# Ensure services have selectors pointing to correct pods
kubectl describe svc multiplication -n microservices

# Verify labels match
kubectl get pods -n microservices -L app
```

### ConfigMap not loading

**Symptom:** Application startup fails trying to load ConfigMap

**Check:**
```bash
# 1. Verify ConfigMap exists
kubectl get configmap -n microservices

# 2. Check ConfigMap content
kubectl describe configmap multiplication-config -n microservices

# 3. Check bootstrap.yml
# In the pod, check if ConfigMap is mounted
kubectl exec -it <pod-name> -n microservices -- ls -la /config

# 4. Check pod logs
kubectl logs -n microservices <pod-name>
```

**Solution:**
```bash
# Apply ConfigMaps
kubectl apply -f k8s/01-configmaps.yaml

# Restart pod to reload
kubectl delete pod <pod-name> -n microservices

# Or restart deployment
kubectl rollout restart deployment/multiplication -n microservices
```

## Resource Issues

### Out of Memory

**Symptom:** `OOMKilled` in events or pod goes to `OOM_KILLED` state

**Check:**
```bash
# Check resource limits
kubectl describe deployment <deployment> -n microservices

# Check actual usage
kubectl top pods -n microservices

# Check events
kubectl get events -n microservices --sort-by='.lastTimestamp'
```

**Solution:**

**For Minikube:**
```bash
# Increase Minikube memory
minikube delete
minikube start --memory=8192 --cpus=4
```

**For deployments - Reduce limits:**
```bash
kubectl set resources deployment/multiplication \
  --limits=memory=512Mi,cpu=500m \
  --requests=memory=256Mi,cpu=250m \
  -n microservices
```

Or edit directly:
```bash
kubectl edit deployment multiplication -n microservices
# Reduce memory limits in spec.template.spec.containers[0].resources
```

### Disk Space Issues

**Check:**
```bash
kubectl top nodes
df -h
```

**Solution:**
```bash
# Clear docker images
docker image prune -a

# For Minikube
minikube ssh -- docker system prune -a
```

## ConfigMap and Secret Issues

### ConfigMap changes not picked up

**Symptom:** Edited ConfigMap but pod still has old values

**Check:**
```bash
# Verify ConfigMap was updated
kubectl get configmap multiplication-config -n microservices -o yaml

# Check if reload is enabled in bootstrap.yml
kubectl logs -n microservices <pod-name> | grep -i reload
```

**Solution:**

**Option 1: Manual restart**
```bash
kubectl rollout restart deployment/multiplication -n microservices
```

**Option 2: Delete pod to force restart**
```bash
kubectl delete pod <pod-name> -n microservices
```

**Option 3: Enable automatic reload (if not already)**
```bash
# Edit bootstrap.yml to include:
# spring:
#   cloud:
#     kubernetes:
#       reload:
#         enabled: true
#         period: 15000ms
```

### Secret not mounted

**Symptom:** Pod can't access environment variables from Secret

**Check:**
```bash
# Verify Secret exists
kubectl get secret -n microservices

# Check pod environment
kubectl exec -it <pod-name> -n microservices -- env | grep <VAR_NAME>

# Check deployment volume mounts
kubectl get deployment <deployment> -n microservices -o yaml | grep -A5 volumeMounts
```

**Solution:**
```bash
# Create Secret
kubectl create secret generic db-credentials \
  --from-literal=password=mypassword \
  -n microservices

# Edit deployment to mount Secret
kubectl edit deployment multiplication -n microservices
# Add to spec.template.spec.containers[0].env:
# - name: DB_PASSWORD
#   valueFrom:
#     secretKeyRef:
#       name: db-credentials
#       key: password
```

## Ingress Issues

### Ingress not working

**Symptom:** Can't access gateway via ingress URL

**Check:**
```bash
# 1. Verify ingress controller is running
kubectl get pods -n ingress-nginx

# 2. Check ingress rules
kubectl get ingress -n microservices
kubectl describe ingress gateway-ingress -n microservices

# 3. Check ingress service
kubectl get svc -n ingress-nginx

# 4. Get ingress IP
kubectl get ingress -n microservices -o wide
```

**Solution:**

**For Minikube:**
```bash
# Enable ingress addon
minikube addons enable ingress

# Get Minikube IP
minikube ip

# Access via IP
curl http://$(minikube ip):8000
```

**For clusters:**
```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

# Wait for LoadBalancer IP
kubectl get svc -n ingress-nginx ingress-nginx-controller -w

# Update /etc/hosts (for local testing)
# <INGRESS_IP> challenges-frontend
```

## Logs and Monitoring

### Viewing logs

```bash
# Tail logs from service
kubectl logs -n microservices -l app=multiplication -f

# Last 50 lines
kubectl logs -n microservices <pod-name> --tail=50

# Previous logs if pod crashed
kubectl logs -n microservices <pod-name> --previous

# All containers in pod
kubectl logs -n microservices <pod-name> --all-containers=true

# Specific timestamp
kubectl logs -n microservices <pod-name> --since=1h

# Search logs for errors
kubectl logs -n microservices -l app=multiplication | grep -i error
```

### Accessing pod shell

```bash
# Interactive shell
kubectl exec -it <pod-name> -n microservices -- /bin/bash

# Run command
kubectl exec <pod-name> -n microservices -- curl http://localhost:8080/actuator/health
```

### Port forwarding for debugging

```bash
# Forward port
kubectl port-forward -n microservices svc/multiplication 8080:8080 &

# Access
curl http://localhost:8080/actuator/env

# Kill port forward
lsof -i :8080
kill -9 <PID>
```

## Common Error Messages

### "unable to connect to the server"

**Solution:**
```bash
# Check cluster connection
kubectl cluster-info

# Verify config
kubectl config view

# Switch context if needed
kubectl config use-context minikube
```

### "no matches for kind"

**Symptom:** Error applying YAML: "no matches for kind"

**Solution:**
```bash
# Check Kubernetes version
kubectl version

# Use correct API version for your K8s version
# Update YAML with correct apiVersion
```

### "connection refused"

**Solution:**
```bash
# Check if service is running
kubectl get pods -n microservices

# Check service endpoints
kubectl get endpoints -n microservices

# Check service ports
kubectl get svc -n microservices -o wide
```

## Kafka-Specific Issues

### Kafka topics not created

**Check:**
```bash
# Check job status
kubectl get job -n microservices

# Check job logs
kubectl logs -n microservices job/kafka-topics-init

# List topics manually
kubectl exec -it <kafka-pod> -n microservices -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Solution:**
```bash
# Create topics manually
kubectl exec -it kafka-0 -n microservices -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic attempts.topic --partitions 4 --replication-factor 1

kubectl exec -it kafka-0 -n microservices -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic logs --partitions 1 --replication-factor 1
```

### Kafka consumer lag

**Check:**
```bash
kubectl exec -it kafka-0 -n microservices -- \
  kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group gamification --describe
```

## Rollback and Reset

### Rollback deployment

```bash
# Check rollout history
kubectl rollout history deployment/multiplication -n microservices

# Rollback to previous version
kubectl rollout undo deployment/multiplication -n microservices

# Rollback to specific revision
kubectl rollout undo deployment/multiplication -n microservices --to-revision=2
```

### Reset everything

```bash
# Delete namespace (all resources)
kubectl delete namespace microservices

# Recreate from scratch
kubectl apply -f k8s/
```

## Performance Issues

### Slow startup

**Check:**
```bash
# Check startup probe times
kubectl describe pod <pod-name> -n microservices

# Check logs for startup messages
kubectl logs -n microservices <pod-name> | head -50
```

**Solution:**
```bash
# Increase startup probe timeout
kubectl edit deployment multiplication -n microservices
# Increase startupProbe.failureThreshold or periodSeconds
```

### High latency

**Check:**
```bash
# Check resource usage
kubectl top pods -n microservices

# Check node resources
kubectl top nodes

# Check network policies
kubectl get networkpolicies -n microservices
```

**Solution:**
```bash
# Scale up deployment
kubectl scale deployment multiplication --replicas=3 -n microservices

# Increase resources
kubectl set resources deployment/multiplication \
  --limits=memory=1Gi,cpu=1000m \
  --requests=memory=512Mi,cpu=500m \
  -n microservices
```

## Quick Reference

```bash
# Essential commands
kubectl get all -n microservices                    # See all resources
kubectl describe pod <pod> -n microservices         # Detailed pod info
kubectl logs -n microservices <pod> -f              # Stream logs
kubectl exec -it <pod> -n microservices -- bash     # Pod shell
kubectl port-forward svc/<service> 8080:8080 -n microservices  # Port forward
kubectl scale deployment <deploy> --replicas=3 -n microservices # Scale
kubectl rollout restart deployment/<deploy> -n microservices    # Restart
kubectl get events -n microservices --sort-by='.lastTimestamp'  # Recent events
```

## Getting Help

1. **Check logs**: `kubectl logs -n microservices <pod>`
2. **Describe resource**: `kubectl describe pod <pod> -n microservices`
3. **Check events**: `kubectl get events -n microservices`
4. **Check documentation**: See README.md, DEPLOYMENT_CHECKLIST.md
5. **Debug with shell**: `kubectl exec -it <pod> -n microservices -- bash`
