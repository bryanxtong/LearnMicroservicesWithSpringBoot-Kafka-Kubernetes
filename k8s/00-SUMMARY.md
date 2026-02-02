# Kubernetes Manifests and Migration Summary

## Overview

This directory contains complete Kubernetes manifests and documentation for deploying your Spring Cloud microservices using **Spring Cloud Kubernetes** instead of Consul. This enables you to run your application on any Kubernetes cluster while removing the dependency on external Consul infrastructure.

## What Was Created

### 1. Kubernetes Manifests (Ready to Deploy)

Located in the `k8s/` directory:

| File | Purpose | Components |
|------|---------|-----------|
| `00-namespace.yaml` | Create `microservices` namespace | Isolated deployment environment |
| `01-configmaps.yaml` | Application configuration | All service configs, shared app config |
| `02-multiplication-deployment.yaml` | Multiplication service | Deployment + Service |
| `03-gamification-deployment.yaml` | Gamification service | Deployment + Service |
| `04-gateway-deployment.yaml` | API Gateway | Deployment + LoadBalancer Service |
| `05-logs-deployment.yaml` | Logging service | Deployment + Service |
| `06-kafka-deployment.yaml` | Message broker | StatefulSet + Topics Job + Services |
| `07-zipkin-deployment.yaml` | Distributed tracing | Deployment + Service |
| `08-ingress.yaml` | External access | NGINX Ingress configuration |
| `09-kafka-ui-deployment.yaml` | Kafka management | Deployment + Service |

**Total:** 10 YAML files covering all infrastructure needs

### 2. Configuration Files for Services

Located in `k8s/config/`:

- `multiplication-bootstrap.yml` - Spring Cloud Kubernetes config loader
- `gamification-bootstrap.yml` - Spring Cloud Kubernetes config loader
- `gateway-bootstrap.yml` - Spring Cloud Kubernetes config loader
- `logs-bootstrap.yml` - Spring Cloud Kubernetes config loader

### 3. Documentation

| Document | Purpose |
|----------|---------|
| `README.md` | Overview and deployment instructions |
| `MIGRATION_GUIDE.md` | pom.xml changes and dependency migration steps |
| `CONFIGMAP_MIGRATION.md` | ConfigMap creation from Consul KV data |
| `DEPLOYMENT_CHECKLIST.md` | Step-by-step deployment verification |
| `QUICKSTART_MINIKUBE.md` | Get running locally in minutes |
| `TROUBLESHOOTING.md` | Common issues and solutions |

## Key Changes from Consul to Kubernetes

### What's Removed
- ✅ **Consul** service discovery → **Kubernetes DNS (built-in)**
- ✅ **Consul** KV store → **Kubernetes ConfigMaps**
- ✅ **Consul** agent registration → **Kubernetes Service discovery**
- ✅ External Consul container → No external dependency

### What's Kept
- ✅ Spring Cloud Gateway (still needed for advanced routing)
- ✅ Kafka (event streaming unchanged)
- ✅ Zipkin (distributed tracing unchanged)
- ✅ H2 databases (per-service data storage)
- ✅ All business logic and APIs

### What's New
- ✅ ConfigMaps for configuration management
- ✅ StatefulSet for Kafka
- ✅ Ingress for external traffic
- ✅ Health probes (liveness, readiness, startup)
- ✅ Resource requests and limits
- ✅ Kubernetes service discovery (automatic)

## Advantages of This Migration

| Aspect | Benefit |
|--------|---------|
| **Simplicity** | Reduce infrastructure: remove Consul, use K8s built-in services |
| **Cost** | No separate Consul nodes, use K8s cluster resources |
| **Scalability** | Automatic service discovery, load balancing |
| **Resilience** | Kubernetes handles pod restarts, scaling, self-healing |
| **Monitoring** | Kubernetes metrics, events, and logging |
| **Operations** | Single tool (kubectl) for all operations |
| **Production Ready** | Works on any Kubernetes cluster (local, cloud, on-prem) |

## Architecture Changes

### Before (with Consul)
```
┌─────────────────────────────────────────────────┐
│          Docker Compose / Kubernetes            │
├─────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────┐              │
│  │ Consul      │  │ Kafka        │              │
│  │ (external)  │  │ (broker)     │              │
│  └─────────────┘  └──────────────┘              │
│         ↑                  ↑                     │
│  ┌──────────────────────────────────────────┐   │
│  │  Microservices                           │   │
│  │  (mult, gamif, gateway, logs)           │   │
│  │  - Service discovery: Consul            │   │
│  │  - Configuration: Consul KV store       │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### After (with Kubernetes)
```
┌──────────────────────────────────────┐
│     Kubernetes Cluster               │
├──────────────────────────────────────┤
│  Built-in                            │
│  ├─ Service Discovery (DNS)          │
│  ├─ Load Balancing                   │
│  ├─ Configuration (ConfigMaps)       │
│  └─ Health Management                │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  Kafka (StatefulSet)           │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  Spring Cloud Microservices    │  │
│  │  (mult, gamif, gateway, logs)  │  │
│  │  - Service discovery: K8s DNS  │  │
│  │  - Configuration: ConfigMaps   │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

## Implementation Steps

### Phase 1: Code Updates (Per Service)

For each microservice (multiplication, gamification, gateway, logs):

**1.1 Update pom.xml**
```xml
<!-- Remove -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>

<!-- Add -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-client</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```

**1.2 Replace bootstrap configuration**
- Delete: `bootstrap.properties`
- Add: Copy file from `k8s/config/*-bootstrap.yml`

**1.3 Clean up application.properties**
- Remove all `spring.cloud.consul.*` properties
- Keep all other properties

**1.4 Build**
```bash
mvn clean package -DskipTests
```

### Phase 2: Infrastructure Setup

**2.1 Build container images**
```bash
# Using Maven
mvn spring-boot:build-image

# Or Docker
docker build -t multiplication:0.0.1-SNAPSHOT multiplication/
```

**2.2 Deploy to Kubernetes**
```bash
# Apply all manifests
kubectl apply -f k8s/

# Or step-by-step for better control
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-configmaps.yaml
kubectl apply -f k8s/06-kafka-deployment.yaml
kubectl apply -f k8s/07-zipkin-deployment.yaml
kubectl apply -f k8s/02-multiplication-deployment.yaml
# ... and so on
```

**2.3 Verify deployment**
```bash
kubectl get all -n microservices
kubectl logs -n microservices -l app=multiplication
```

## Configuration Hierarchy

Spring Cloud Kubernetes loads configuration in this order (first wins):

1. **Application bootstrap.yml** (in classpath)
2. **Service-specific ConfigMap** (e.g., `multiplication-config`)
3. **Shared ConfigMap** (named `application`)
4. **Secrets** (if configured)

Example for Multiplication service:
```
bootstrap.yml (Spring Cloud Kubernetes settings)
    ↓
multiplication-config ConfigMap (service-specific props)
    ↓
application ConfigMap (shared props like logging)
    ↓
Environment variables (if needed)
```

## ConfigMaps Provided

### `application` (shared)
```yaml
logging:
  level:
    org.springframework.core.env: DEBUG
```

### `multiplication-config`
- Database: H2 file-based
- Kafka: Bootstrap servers, partitions, replicas
- Tracing: Zipkin endpoint
- Logging: Pattern and levels

### `gamification-config`
- Database: H2 file-based
- Kafka: Consumer settings, group ID
- Tracing: Zipkin endpoint
- Logging: Pattern and levels

### `gateway-config`
- Gateway: Routes to multiplication and gamification
- CORS: Allows localhost:3000 and challenges-frontend:3000
- Retry logic: 3 retries for GET and POST
- Observability: Tracing enabled

### `logs-config`
- Kafka consumer: Group ID, auto-offset-reset
- Ports and endpoints

## Quick Deployment

### For Minikube (Local Development)

```bash
# 1. Start Minikube
minikube start --memory=4096 --cpus=4

# 2. Enable ingress
minikube addons enable ingress

# 3. Build images
mvn clean package -DskipTests  # Each service
minikube image load <image-name>

# 4. Deploy
kubectl apply -f k8s/

# 5. Access
kubectl port-forward -n microservices svc/gateway 8000:8000
# http://localhost:8000
```

### For Cloud Kubernetes (EKS, GKE, AKS)

```bash
# 1. Configure kubectl for your cluster
kubectl config use-context <your-context>

# 2. Push images to registry
docker push <registry>/multiplication:0.0.1-SNAPSHOT
docker push <registry>/gamification:0.0.1-SNAPSHOT
docker push <registry>/gateway:0.0.1-SNAPSHOT
docker push <registry>/logs:0.0.1-SNAPSHOT

# 3. Update image references in manifests
# Change imagePullPolicy to "Always"
# Update image paths to your registry

# 4. Deploy
kubectl apply -f k8s/

# 5. Get external IP
kubectl get svc gateway -n microservices
```

## Service Discovery

### How it works in Kubernetes

No need to configure anything - it's automatic!

**Service DNS:** `<service-name>.<namespace>.svc.cluster.local`

Examples:
- `multiplication.microservices.svc.cluster.local:8080`
- `kafka-broker.microservices.svc.cluster.local:9092`
- `zipkin.microservices.svc.cluster.local:9411`

Gateway automatically discovers services via Spring Cloud Kubernetes:
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: multiplication
          uri: lb://multiplication/  # Kubernetes DNS resolution
          predicates:
            - Path=/challenges/**,/attempts/**,/users/**
```

## Environment Variables in Deployments

All services receive these environment variables:

```yaml
env:
- name: spring.application.name
  value: "multiplication"
- name: SPRING_CLOUD_KAFKA_HOST
  value: "kafka-broker"
- name: SPRING_CLOUD_ZIPKIN_HOST
  value: "zipkin"
- name: SPRING_KAFKA_BOOTSTRAP_SERVERS
  value: "kafka-broker:9092"
- name: MANAGEMENT_TRACING_EXPORT_ZIPKIN_ENDPOINT
  value: "http://zipkin:9411/api/v2/spans"
```

These override default values and point to Kubernetes service DNS names.

## Scaling

### Horizontal Scaling

```bash
# Scale multiplication to 3 replicas
kubectl scale deployment multiplication --replicas=3 -n microservices

# Verify
kubectl get deployment multiplication -n microservices
kubectl get pods -n microservices -l app=multiplication
```

Kubernetes automatically:
- Load balances requests across replicas
- Distributes pods across nodes
- Restarts failed pods
- Updates services automatically

### Vertical Scaling (Resource Limits)

```bash
# Increase resource limits
kubectl set resources deployment multiplication \
  --limits=memory=1Gi,cpu=1000m \
  --requests=memory=512Mi,cpu=500m \
  -n microservices
```

## Monitoring and Observability

### Health Checks
All deployments include:
- **Liveness probe:** Detects dead pods
- **Readiness probe:** Waits for startup
- **Startup probe:** Grace period for app startup

### Metrics
Available at `/actuator/prometheus` on each service:
```bash
curl http://localhost:8000/actuator/prometheus
```

### Tracing
Access Zipkin for distributed traces:
```bash
kubectl port-forward -n microservices svc/zipkin 9411:9411
# http://localhost:9411
```

### Logs
Centralized through Kubernetes:
```bash
# All logs from multiplication
kubectl logs -n microservices -l app=multiplication -f

# Pod shell for debugging
kubectl exec -it <pod-name> -n microservices -- bash
```

## What's NOT Included (Future Work)

These are optional enhancements:

- [ ] Persistent storage for databases (currently H2 in-pod)
- [ ] YAML templating with Helm or Kustomize
- [ ] Network policies for security
- [ ] Pod Disruption Budgets for availability
- [ ] Horizontal Pod Autoscaling (HPA)
- [ ] Service Mesh (Istio, Linkerd)
- [ ] Certificate management (cert-manager)
- [ ] ArgoCD for GitOps deployment
- [ ] Prometheus/Grafana for metrics
- [ ] ELK/Loki for centralized logging

## File Summary

```
k8s/
├── 00-namespace.yaml              # Kubernetes namespace
├── 01-configmaps.yaml             # Application configuration
├── 02-multiplication-deployment.yaml
├── 03-gamification-deployment.yaml
├── 04-gateway-deployment.yaml
├── 05-logs-deployment.yaml
├── 06-kafka-deployment.yaml       # Kafka StatefulSet + topics
├── 07-zipkin-deployment.yaml
├── 08-ingress.yaml
├── 09-kafka-ui-deployment.yaml
├── config/
│   ├── multiplication-bootstrap.yml
│   ├── gamification-bootstrap.yml
│   ├── gateway-bootstrap.yml
│   └── logs-bootstrap.yml
├── README.md                       # Full deployment guide
├── QUICKSTART_MINIKUBE.md         # Local development
├── DEPLOYMENT_CHECKLIST.md        # Step-by-step verification
├── MIGRATION_GUIDE.md             # Code changes needed
├── CONFIGMAP_MIGRATION.md         # Configuration management
└── TROUBLESHOOTING.md             # Common issues & solutions
```

## Next Steps

1. **Read:** Start with `README.md` for full overview
2. **Prepare:** Use `MIGRATION_GUIDE.md` to update your code
3. **Test Locally:** Follow `QUICKSTART_MINIKUBE.md` with Minikube
4. **Deploy:** Use `DEPLOYMENT_CHECKLIST.md` for verification
5. **Troubleshoot:** Refer to `TROUBLESHOOTING.md` if issues arise

## Key Takeaways

✅ **Complete K8s manifests** ready to deploy
✅ **Removed Consul dependency** - simpler, cheaper, more maintainable
✅ **Kept Spring Cloud Gateway** - still needed for advanced routing
✅ **Preserved all functionality** - same business logic, better infrastructure
✅ **Production ready** - works on any Kubernetes cluster
✅ **Well documented** - guides for every stage of deployment
✅ **Easy troubleshooting** - comprehensive troubleshooting guide included

## Support Resources

- **Kubernetes Docs:** https://kubernetes.io/docs/
- **Spring Cloud Kubernetes:** https://spring.io/projects/spring-cloud-kubernetes
- **kubectl Cheat Sheet:** https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Minikube:** https://minikube.sigs.k8s.io/

---

**You're all set!** Start with the QUICKSTART_MINIKUBE.md or README.md based on your target environment.
