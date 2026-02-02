# pom.xml Changes for Spring Cloud Kubernetes Migration

## Remove These Dependencies (Consul)

```xml
<!-- Remove from all services -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-config</artifactId>
</dependency>
```

## Add These Dependencies (Kubernetes)

```xml
<!-- Add to all services -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>

<!-- Optional: For bootstrap ConfigMap loading (recommended) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-kubernetes-client-bootstrap</artifactId>
</dependency>
```

## Example Complete pom.xml Section

```xml
<properties>
    <spring-cloud.version>2025.1.0</spring-cloud.version>
</properties>

<dependencies>
    <!-- Spring Cloud Kubernetes (replaces Consul) -->
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

    <!-- Keep existing dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-zipkin</artifactId>
    </dependency>
    <!-- ... other dependencies ... -->
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## Migration Steps by Service

### 1. For Each Service (multiplication, gamification, gateway, logs)

**Step 1: Update pom.xml**
- Remove: `spring-cloud-starter-consul-discovery`
- Remove: `spring-cloud-starter-consul-config`
- Add: `spring-cloud-starter-kubernetes-client`
- Add: `spring-cloud-starter-kubernetes-client-config`
- Add: `spring-cloud-kubernetes-client-bootstrap`

**Step 2: Replace bootstrap.properties with bootstrap.yml**
- Delete: `bootstrap.properties`
- Create: `bootstrap.yml` (use files in k8s/config/ directory)

**Step 3: Clean up application.properties**
- Remove: `spring.cloud.consul.*` properties
- Keep: Kafka, Zipkin, application-specific properties

**Step 4: Remove Consul config import**
- Remove: `spring.config.import=optional:consul:` (if present)

### 2. Rebuild and Test

```bash
# Clean and build each service
mvn clean package -DskipTests

# Or with Docker
mvn spring-boot:build-image
```

## Version Compatibility

Ensure you have the right versions:
- Spring Boot: 4.0.1
- Spring Cloud: 2025.1.0
- Java: 17+

## ConfigMap Format

The bootstrap.yml will automatically load:

1. **ConfigMap named after service**: e.g., `multiplication-config`
2. **Shared ConfigMap**: named `application`

Configuration is loaded from ConfigMaps in this order (first wins):
- ConfigMap with service name
- ConfigMap named `application`

## Notes

- No need to set `spring.config.import` anymore (Spring Cloud Kubernetes auto-configures)
- Kubernetes namespace must match the one specified in bootstrap.yml (default: `microservices`)
- For local development with Minikube, ensure `kubectl config current-context` points to minikube
- ConfigMap changes can be auto-reloaded if enabled in bootstrap.yml (as configured above)
