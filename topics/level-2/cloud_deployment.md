### Topic: Cloud Deployment — Level 2

> **Related:** [Level 2 — CI/CD](./ci_cd.md) · [Level 2 — Monitoring and Alerting](./monitoring_and_alerting.md) · [Level 2 — Microservices](./microservices.md)

**Why it matters (Karat angle)**
Citi deploys on OpenShift (Kubernetes-based). Interviewers ask "how do you deploy a Spring Boot app?" expecting Docker, Kubernetes, and cloud-native concepts — not "I run `java -jar`."

**Core concept**

**Deployment pipeline:**
```
Code → Build (Maven/Gradle) → Docker image → Push to registry → Deploy to Kubernetes/OpenShift
```

**Dockerizing a Spring Boot app:**

```dockerfile
# File: topics/level-2/Dockerfile

# Multi-stage build — keeps image small
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

# Non-root user (security best practice)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
# Build and run
docker build -t payment-service:1.0 .
docker run -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod payment-service:1.0
```

**Kubernetes deployment:**

```yaml
# File: topics/level-2/k8s-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        - name: payment-service
          image: registry.citi.com/payment-service:1.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

**Key Kubernetes concepts:**

| Concept | What it is |
|---------|-----------|
| **Pod** | Smallest deployable unit (one or more containers) |
| **Deployment** | Manages pod replicas, rolling updates |
| **Service** | Stable network endpoint for pods |
| **ConfigMap** | Non-sensitive config (properties, YAML) |
| **Secret** | Sensitive config (passwords, keys) — base64 encoded |
| **Ingress** | HTTP routing from external traffic to services |
| **HPA** | Horizontal Pod Autoscaler — scale based on CPU/memory |

**Spring Boot + Kubernetes integration:**

```yaml
# application.yml
spring:
  config:
    import: "optional:configtree:/etc/config/"  # Kubernetes ConfigMap mounted as files

management:
  endpoint:
    health:
      probes:
        enabled: true                           # enables /liveness and /readiness
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

**What to say in the interview (4-beat answer)**
1. **Definition:** Cloud deployment uses Docker containers on Kubernetes (or OpenShift). Spring Boot apps are packaged as Docker images, deployed as Kubernetes pods with health probes, resource limits, and auto-scaling.
2. **Why/when:** Containers ensure consistent environments (dev = prod). Kubernetes handles scaling, self-healing (restart crashed pods), rolling updates, and service discovery.
3. **Example:** A Deployment with 3 replicas, liveness probe on `/actuator/health/liveness`, readiness probe on `/actuator/health/readiness`, DB password from a Kubernetes Secret, and HPA scaling from 3 to 10 pods based on CPU.
4. **Gotcha/tradeoff:** Running as root in a container is a security risk — always use a non-root user. JVM in a container needs `-XX:MaxRAMPercentage=75` to respect container memory limits (not host memory).

**Common pitfalls**
- JVM not respecting container memory limits — use `-XX:MaxRAMPercentage` or set `-Xmx` explicitly.
- Running as root in Docker — security vulnerability; use `USER appuser`.
- No readiness probe — Kubernetes sends traffic to pods before they're ready.
- Storing secrets in ConfigMaps — use Secrets (still base64, not encrypted by default; use a vault for real security).
- Large Docker images — use multi-stage builds and JRE (not JDK) for production.

**Self-check question**
Your Spring Boot app takes 60 seconds to start. Kubernetes liveness probe has `initialDelaySeconds=10` and `failureThreshold=3`. What happens to your pod? How do you fix it?
