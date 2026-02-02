# ConfigMap Migration from Consul

## Current Consul Configuration

Your current Consul KV configuration contains:

```
config/application,docker/data: (base64 encoded)

Decoded content:
logging:
  level:
    org.springframework.core.env: DEBUG
```

## Kubernetes ConfigMap Equivalents

The provided ConfigMaps in `01-configmaps.yaml` already include this configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: application
  namespace: microservices
data:
  application.yml: |
    logging:
      level:
        org.springframework.core.env: DEBUG
```

## How Spring Cloud Kubernetes Loads Configuration

### 1. Bootstrap Phase
Spring Cloud Kubernetes loads configuration from:
- `bootstrap.yml` or `bootstrap.properties` in classpath (existing app config)
- ConfigMap matching namespace and application name

### 2. ConfigMap Discovery Order
For a service named `multiplication`:
1. Look for ConfigMap: `multiplication-config` in namespace `microservices`
2. Look for ConfigMap: `application` in namespace `microservices`
3. Merge both (service-specific wins on conflicts)

### 3. Configuration Properties Format

**For .properties files in ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: multiplication-config
data:
  application.properties: |
    spring.datasource.url=jdbc:h2:file:/tmp/multiplication
    kafka.attempts=attempts.topic
```

**For .yml files in ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: multiplication-config
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:h2:file:/tmp/multiplication
    kafka:
      attempts: attempts.topic
```

## Adding New Configuration

### Via kubectl

```bash
# Edit a ConfigMap
kubectl edit configmap multiplication-config -n microservices

# Or create from file
kubectl create configmap my-config --from-file=application.properties -n microservices
```

### Via YAML

```bash
# Apply updated ConfigMap
kubectl apply -f 01-configmaps.yaml
```

### Configuration Reload

Services with Spring Cloud Kubernetes reload enabled will detect ConfigMap changes automatically:

```yaml
spring:
  cloud:
    kubernetes:
      reload:
        enabled: true
        mode: polling
        period: 15000ms  # Check every 15 seconds
```

After reload, properties are available through `@ConfigurationProperties` and `@Value` annotations.

## Secrets vs ConfigMaps

For sensitive data like API keys, database passwords, etc., use Kubernetes Secrets instead:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: microservices
type: Opaque
stringData:
  username: admin
  password: secret123
```

Then reference in bootstrap.yml:
```yaml
spring:
  cloud:
    kubernetes:
      secrets:
        enabled: true
        sources:
          - name: database-credentials
```

## Comparison: Consul vs Kubernetes ConfigMaps

| Aspect | Consul | Kubernetes ConfigMaps |
|--------|--------|----------------------|
| **Storage** | External KV store | Native K8s resource |
| **Hierarchy** | key/value path | key: data |
| **Max Size** | 512KB per value | 1MB per ConfigMap |
| **Updates** | Requires watcher service | Native K8s watch |
| **Encryption** | Optional with TLS | Secrets for sensitive data |
| **Multi-tenant** | Namespaces | K8s namespaces |
| **Tool** | `consul` CLI | `kubectl` |

## Migration Checklist

- [ ] Create `01-configmaps.yaml` with all necessary configs
- [ ] Create service-specific ConfigMaps (multiplication-config, gamification-config, etc.)
- [ ] Create `application` shared ConfigMap
- [ ] Update each service's `bootstrap.yml`
- [ ] Remove `bootstrap.properties` Consul references
- [ ] Update `pom.xml` dependencies (remove Consul, add Kubernetes)
- [ ] Test configuration loading with `kubectl logs`
- [ ] Verify services start correctly
- [ ] Delete Consul from docker-compose.yml (optional for local dev)

## Example: Adding Database Configuration

If you need to add database configuration:

1. **Add to ConfigMap:**
```yaml
kubectl edit configmap multiplication-config -n microservices
```

2. **Add property:**
```properties
spring.datasource.password=mypassword
spring.datasource.username=myuser
```

3. **Pod automatically reloads** (if reload enabled)

Or **restart pod** to pick up changes:
```bash
kubectl rollout restart deployment/multiplication -n microservices
```

## Troubleshooting ConfigMap Loading

### Check if ConfigMap exists
```bash
kubectl get configmaps -n microservices
kubectl describe configmap multiplication-config -n microservices
```

### Check pod logs for configuration errors
```bash
kubectl logs deployment/multiplication -n microservices
```

### Verify configuration was loaded
```bash
# Access the pod
kubectl exec -it pod/multiplication-xxx -n microservices -- /bin/sh

# Check system properties
curl http://localhost:8080/actuator/env
```

### Verify reload is working
```bash
# Watch logs while editing ConfigMap
kubectl logs deployment/multiplication -n microservices -f

# In another terminal, edit the ConfigMap
kubectl edit configmap multiplication-config -n microservices

# Should see reload messages in logs
```
