# Manual K8s Deployment Guide with Sealed Secrets

This document outlines the complete step-by-step process of deploying the `product-service` to a local Kubernetes cluster (namespace: `dev`), including Docker image building, K8s manifests, Sealed Secrets management, and troubleshooting.

## Prerequisites
- **Docker Desktop** (for building and pushing images)
- **kubectl** (configured with cluster access)
- **Helm** (for Sealed Secrets controller)
- **kubeseal CLI** (for encrypting secrets)
- **Docker Hub Account** (`username`)

---

## Phase 1: Containerization (Docker)

### 1.1 Dockerfile
Create a multi-stage `Dockerfile` in the `product-service` root directory:

```dockerfile
# Stage 1: Build
FROM gradle:8.5-jdk21 AS build
WORKDIR /app
COPY build.gradle settings.gradle ./
COPY src ./src
RUN gradle clean bootJar -x test --no-daemon

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

ENV TZ=UTC
ENV JAVA_TOOL_OPTIONS="-Duser.timezone=UTC -XX:MaxRAMPercentage=75"

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /app/build/libs/*.jar app.jar

EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 1.2 Build & Push Image
```bash
cd product-service

# Build
docker build -t username/product-service:1.0.0 .

# Push
docker push username/product-service:1.0.0
```

---

## Phase 2: Kubernetes Manifests

Create the following files in the `k8s-gitops/services/product-service/` directory.

### 2.1 ConfigMap (`configmap.yaml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-service-config
  namespace: dev
data:
  SPRING_PROFILES_ACTIVE: "dev"
  DB_HOST: "x.x.x.231"
  DB_PORT: "5432"
  DB_NAME: "product_db"
  DB_USER: "postgres"
  REDIS_SENTINEL_1: "x.x.x.236:26379"
  REDIS_SENTINEL_2: "x.x.x.236:26380"
  REDIS_SENTINEL_3: "x.x.x.236:26381"
```

### 2.2 Deployment (`deployment.yaml`)
```yaml
piVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
        - name: product-service
          image: username/product-service:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8081
          envFrom:
            - configMapRef:
                name: product-service-config
            - secretRef:
                name: product-service-secret # add secret referene
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8081 }
            initialDelaySeconds: 15


```

### 2.3 Service (`service.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: dev
spec:
  selector:
    app: product-service
  ports:
    - port: 8081
      targetPort: 8081
```

### 2.4 Ingress (`ingress.yaml`)
*Note: `host` is removed to allow access via raw IP address.*
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: product-service
  namespace: dev
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
    - http:   # must dash (-)
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8081

```

---

## Phase 3: Secret Management (Sealed Secrets)

### 3.1 Install Controller (K8s)
```bash
helm install sealed-secrets oci://registry-1.docker.io/bitnamicharts/sealed-secrets -n kube-system
```

### 3.2 Install `kubeseal` CLI (Laptop)
```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/kubeseal-0.27.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.27.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

### 3.3 Encrypt the Secret (Admin Task)
Create `secret-plain.yaml` (DO NOT COMMIT THIS FILE):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: product-service-secret
  namespace: dev
type: Opaque
stringData:
  DB_PASSWORD: "123"
  REDIS_PASSWORD: "123"
```

Encrypt it:
```bash
kubeseal --controller-name sealed-secrets --controller-namespace kube-system --format yaml < secret-plain.yaml > sealed-secret.yaml
```
Move `sealed-secret.yaml` to `k8s-gitops/services/product-service/` and delete `secret-plain.yaml`.

---

## Phase 4: Manual Deployment

```bash
# Create namespace
kubectl create namespace dev

# Apply all manifests
cd k8s-gitops/services/product-service
kubectl apply -f .
```

### Verification
```bash
# Check Pods
kubectl -n dev get pods -w

# Check Sealed Secret (should show Unsealed)
kubectl -n dev get sealedsecret
kubectl -n dev get secret product-service-secret

# Check Logs (dev profile)
kubectl -n dev logs -l app=product-service --tail=30
```

---

## New Version Image build and deploy

```
cd ~/Documents/micro-services/product-service

# new image build and push
docker build -t alhajcse/product-service:1.5.0 .
docker push alhajcse/product-service:1.5.0


cd ~/Documents/k8s-gitops/services/product-service
# Update yaml file
nano deployment.yaml

# ─── Deployment update
kubectl apply -f deployment.yaml

# ─── Rollout Status ───
kubectl rollout status deploy/product-service -n dev
# → deployment "product-service" successfully rolled out

# ─── Pods (New image) ───
kubectl get pods -n dev -l app=product-service -o wide
# → 2 pods Running (image: alhajcse/product-service:1.4.0)

Or

# update K8s
kubectl -n dev set image deployment/product-service product-service=alhajcse/product-service:1.5.0

# wait for Pod ready
kubectl -n dev get pods -w

```


## Phase 5: Testing & Troubleshooting

### API Testing via Traefik IP
```bash
curl -X 'GET' \
  'http://10.70.16.237:30080/api/v1/products?category=ELECTRONICS&page=0&size=20&from=createdAt&order=desc' \
  -H 'accept: application/json'
```

### Troubleshooting (Issues Fixed)

#### Issue 1: CORS / `Failed to fetch` in Swagger UI
**Cause:** Swagger UI tried to hit `http://localhost:8081` instead of the public IP.
**Fix:** 
1. Update `OpenApiConfig.java` to use relative URL: `.servers(List.of(new Server().url("/")))`.
2. Add `WebConfig.java` to allow CORS from all origins.
3. Set `disable-swagger-default-url: true` in `application.yml`.


---

**Next Steps:** Push all these manifests to the `k8s-gitops` Git repository and set up ArgoCD for automated GitOps deployment.
