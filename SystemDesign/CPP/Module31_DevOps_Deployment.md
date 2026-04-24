# Module 31: DevOps & Deployment

> DevOps bridges the gap between development and operations — enabling teams to build, test, and deploy software faster and more reliably. In system design, understanding containerization, orchestration, CI/CD pipelines, deployment strategies, infrastructure as code, and cloud services is essential for designing systems that can be operated at scale. This module covers the tools and practices that turn code into running, reliable production systems.

---

## 31.1 Containerization

> **Containers** package an application and all its dependencies (libraries, runtime, configuration) into a single, portable unit that runs consistently across any environment. Docker is the standard container runtime, and containers are the building blocks of modern microservices deployments.

---

### Docker Basics

**What is a Container?**

```
Traditional Deployment:              Container Deployment:
+---------------------------+        +---------------------------+
| App A  | App B  | App C   |        | Container A | Container B|
|--------|--------|---------|        |  App A      |  App B     |
| Libs A | Libs B | Libs C  |        |  Libs A     |  Libs B    |
|--------|--------|---------|        |  Runtime A  |  Runtime B |
|     Operating System      |        +-------------|------------+
|     (shared, conflicts!)  |        |     Container Runtime    |
+---------------------------+        |     (Docker Engine)      |
|       Hardware            |        +---------------------------+
+---------------------------+        |     Operating System      |
                                     +---------------------------+
                                     |       Hardware            |
                                     +---------------------------+

Traditional: Apps share OS and libraries → dependency conflicts!
Containers: Each app has its own isolated environment → no conflicts!
```

**Container vs Virtual Machine:**

| Feature | Container | Virtual Machine |
|---------|-----------|----------------|
| **Isolation** | Process-level (shared kernel) | Hardware-level (separate kernel) |
| **Startup time** | Seconds | Minutes |
| **Size** | MBs (10-500 MB) | GBs (1-20 GB) |
| **Resource overhead** | Minimal (shared kernel) | Significant (full OS per VM) |
| **Density** | 100s per host | 10s per host |
| **Portability** | Excellent (runs anywhere Docker runs) | Good (requires hypervisor) |
| **Security** | Weaker isolation (shared kernel) | Stronger isolation (separate kernel) |
| **Use case** | Microservices, CI/CD, development | Multi-tenant, different OS requirements |

---

### Dockerfile

A **Dockerfile** is a text file with instructions for building a Docker image.

```dockerfile
# Multi-stage build for a Go application

# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download                    # cache dependencies
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# Stage 2: Runtime (minimal image)
FROM alpine:3.19
RUN apk --no-cache add ca-certificates  # for HTTPS calls
COPY --from=builder /app/server /server
EXPOSE 8080
USER nonroot:nonroot                    # don't run as root!
ENTRYPOINT ["/server"]
```

**Dockerfile Best Practices:**

| Practice | Why |
|----------|-----|
| **Use multi-stage builds** | Build in a large image (with compilers), copy only the binary to a small runtime image |
| **Use specific base image tags** | `golang:1.22-alpine` not `golang:latest` — reproducible builds |
| **Minimize layers** | Combine RUN commands; each layer adds size |
| **Order by change frequency** | Put rarely-changing instructions first (dependencies before source code) — better layer caching |
| **Don't run as root** | Use `USER nonroot` — limits damage if container is compromised |
| **Use .dockerignore** | Exclude unnecessary files (node_modules, .git, tests) from the build context |
| **Pin dependency versions** | Reproducible builds; avoid surprises from updated dependencies |
| **Use small base images** | `alpine` (5 MB) or `distroless` (2 MB) instead of `ubuntu` (72 MB) |

---

### Docker Compose

**Docker Compose** defines and runs multi-container applications using a YAML file. Ideal for local development and testing.

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: ./api
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  worker:
    build: ./worker
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache

volumes:
  postgres_data:
```

---

### Container Registries

A **container registry** stores and distributes Docker images.

| Registry | Type | Key Features |
|----------|------|-------------|
| **Docker Hub** | Public/Private | Default registry; largest public image library |
| **Amazon ECR** | AWS managed | Integrated with ECS/EKS; image scanning; lifecycle policies |
| **Google Artifact Registry** | GCP managed | Multi-format (Docker, Maven, npm); integrated with GKE |
| **Azure Container Registry** | Azure managed | Integrated with AKS; geo-replication |
| **GitHub Container Registry** | GitHub managed | Integrated with GitHub Actions; free for public images |
| **Harbor** | Open-source | Self-hosted; vulnerability scanning; RBAC; replication |

**Image Tagging Strategy:**

```
❌ Bad: myapp:latest (mutable — what version is "latest"?)

✅ Good:
  myapp:1.2.3              (semantic version)
  myapp:abc123def           (git commit SHA — immutable, traceable)
  myapp:1.2.3-abc123def    (version + commit — best of both)
  myapp:main-20250315-001  (branch + date + build number)
```

---

### Multi-Stage Builds

**Multi-stage builds** use multiple `FROM` statements to create smaller, more secure production images.

```
Without Multi-Stage:
  Build image: golang:1.22 (800 MB) + source code + build tools
  → Final image: 800+ MB (includes compiler, source code, build tools — unnecessary in production!)

With Multi-Stage:
  Stage 1 (builder): golang:1.22 (800 MB) → compile → binary (15 MB)
  Stage 2 (runtime): alpine:3.19 (5 MB) + binary (15 MB)
  → Final image: 20 MB (97.5% smaller!)
```

**Multi-Stage for Different Languages:**

| Language | Builder Image | Runtime Image | Typical Final Size |
|----------|-------------|--------------|-------------------|
| Go | golang:1.22 | alpine or scratch | 10-30 MB |
| Java | maven:3.9 or gradle:8 | eclipse-temurin:21-jre-alpine | 150-300 MB |
| Node.js | node:20 | node:20-alpine | 100-200 MB |
| Python | python:3.12 | python:3.12-slim | 100-200 MB |
| Rust | rust:1.77 | alpine or scratch | 5-20 MB |
| C++ | gcc:13 or cmake | alpine or scratch | 5-30 MB |

---


## 31.2 Orchestration

> **Container orchestration** automates the deployment, scaling, networking, and management of containerized applications across a cluster of machines. **Kubernetes** (K8s) is the industry standard for container orchestration.

---

### Kubernetes Basics

```
Kubernetes Cluster Architecture:

  +------------------------------------------------------------------+
  | Control Plane (Master)                                           |
  |  +-------------+  +-------------+  +----------+  +----------+   |
  |  | API Server  |  | Scheduler   |  | Controller|  | etcd     |   |
  |  | (kubectl    |  | (assigns    |  | Manager   |  | (cluster |   |
  |  |  talks here)|  |  pods to    |  | (maintains|  |  state)  |   |
  |  |             |  |  nodes)     |  |  desired  |  |          |   |
  |  +-------------+  +-------------+  |  state)   |  +----------+   |
  |                                    +----------+                  |
  +------------------------------------------------------------------+
  
  +---------------------------+  +---------------------------+
  | Worker Node 1             |  | Worker Node 2             |
  |  +-------+  +-------+    |  |  +-------+  +-------+    |
  |  | Pod A |  | Pod B |    |  |  | Pod C |  | Pod D |    |
  |  | (app) |  | (app) |    |  |  | (app) |  | (db)  |    |
  |  +-------+  +-------+    |  |  +-------+  +-------+    |
  |  +---------------------+  |  |  +---------------------+  |
  |  | kubelet | kube-proxy|  |  |  | kubelet | kube-proxy|  |
  |  +---------------------+  |  |  +---------------------+  |
  +---------------------------+  +---------------------------+
```

---

### Core Kubernetes Objects

**Pod:**
The smallest deployable unit in Kubernetes. A pod contains one or more containers that share networking and storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  labels:
    app: api
spec:
  containers:
    - name: api
      image: myapp/api:1.2.3
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"        # 0.25 CPU cores (minimum guaranteed)
          memory: "256Mi"    # 256 MB (minimum guaranteed)
        limits:
          cpu: "500m"        # 0.5 CPU cores (maximum allowed)
          memory: "512Mi"    # 512 MB (maximum — OOMKilled if exceeded)
      livenessProbe:
        httpGet:
          path: /health/live
          port: 8080
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        periodSeconds: 5
```

**Deployment:**
Manages a set of identical pods (ReplicaSet). Handles rolling updates, rollbacks, and scaling.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3                          # run 3 identical pods
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                      # max 1 extra pod during update
      maxUnavailable: 0                # never have fewer than 3 pods
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myapp/api:1.2.3
          ports:
            - containerPort: 8080
```

**Service:**
A stable network endpoint that routes traffic to a set of pods (selected by labels). Pods come and go; the Service IP stays the same.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api                           # routes to pods with label app=api
  ports:
    - port: 80                         # service port (what clients connect to)
      targetPort: 8080                 # pod port (where the app listens)
  type: ClusterIP                      # internal only (default)
```

**Service Types:**

| Type | Description | Use Case |
|------|-------------|----------|
| **ClusterIP** | Internal IP only (default) | Service-to-service communication |
| **NodePort** | Exposes on each node's IP at a static port (30000-32767) | Development, testing |
| **LoadBalancer** | Provisions a cloud load balancer (ALB/NLB) | External traffic |
| **ExternalName** | Maps to an external DNS name (CNAME) | External service integration |

---

### ConfigMaps and Secrets

**ConfigMap:** Stores non-sensitive configuration data as key-value pairs.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "db.example.com"
  LOG_LEVEL: "info"
  FEATURE_FLAGS: |
    {"dark_mode": true, "new_checkout": false}
```

**Secret:** Stores sensitive data (passwords, tokens, keys). Base64-encoded (NOT encrypted by default — use external secrets management for real security).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # base64("admin")
  password: c3VwZXJzZWNyZXQ=  # base64("supersecret")
```

**Using ConfigMaps and Secrets in Pods:**

```yaml
spec:
  containers:
    - name: api
      image: myapp/api:1.2.3
      envFrom:
        - configMapRef:
            name: app-config           # all keys as env vars
        - secretRef:
            name: db-credentials       # all keys as env vars
      env:
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password            # single key as env var
```

**External Secrets Management:**
For production, use an external secrets manager instead of Kubernetes Secrets:
- **External Secrets Operator** — syncs secrets from Vault/AWS Secrets Manager/GCP Secret Manager into K8s Secrets
- **Sealed Secrets** — encrypts secrets so they can be safely stored in Git
- **HashiCorp Vault** — inject secrets directly into pods via sidecar or CSI driver

---

### Ingress Controllers

An **Ingress** defines rules for routing external HTTP/HTTPS traffic to internal services. An **Ingress Controller** (Nginx, Traefik, AWS ALB) implements these rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

```
External Traffic Flow:

  Internet → DNS → Cloud LB → Ingress Controller (Nginx)
                                  ├── /api/users  → user-service (pods)
                                  ├── /api/orders → order-service (pods)
                                  └── /           → frontend (pods)
```

---

### Namespaces

**Namespaces** provide logical isolation within a cluster — separate environments, teams, or applications.

```
Cluster:
  ├── namespace: production
  │   ├── deployment: api-server (3 replicas)
  │   ├── deployment: worker (5 replicas)
  │   └── service: api-service
  ├── namespace: staging
  │   ├── deployment: api-server (1 replica)
  │   └── service: api-service
  ├── namespace: monitoring
  │   ├── deployment: prometheus
  │   ├── deployment: grafana
  │   └── deployment: loki
  └── namespace: kube-system
      ├── deployment: coredns
      ├── deployment: kube-proxy
      └── daemonset: aws-node
```

**Namespace Use Cases:**
- **Environment separation:** production, staging, development
- **Team isolation:** team-a, team-b (with RBAC per namespace)
- **Application isolation:** app-1, app-2
- **Resource quotas:** limit CPU/memory per namespace

---


## 31.3 CI/CD

> **CI/CD** (Continuous Integration / Continuous Delivery / Continuous Deployment) automates the process of building, testing, and deploying software. It's the backbone of modern software delivery — enabling teams to ship changes multiple times per day with confidence.

---

### Continuous Integration (CI)

**CI** is the practice of frequently merging code changes into a shared repository, with each merge triggering an automated build and test pipeline.

```
CI Pipeline:

  Developer pushes code → Trigger CI Pipeline
    ↓
  1. Checkout code
  2. Install dependencies
  3. Lint / Static analysis
  4. Compile / Build
  5. Run unit tests
  6. Run integration tests
  7. Build Docker image
  8. Push image to registry
  9. Report results (pass/fail)
    ↓
  If ALL steps pass → code is ready for deployment
  If ANY step fails → developer is notified → fix and re-push
```

**CI Best Practices:**

| Practice | Why |
|----------|-----|
| **Run on every push/PR** | Catch issues early, before they reach main branch |
| **Fast feedback** (< 10 minutes) | Developers wait for results; slow CI → slow development |
| **Fail fast** | Run linting and unit tests first (fast); integration tests later (slow) |
| **Reproducible builds** | Pin dependency versions; use Docker for consistent build environment |
| **Branch protection** | Require CI to pass before merging to main |
| **Parallel execution** | Run independent test suites in parallel to reduce total time |

---

### Continuous Delivery vs Continuous Deployment

| Feature | Continuous Delivery | Continuous Deployment |
|---------|-------------------|---------------------|
| **Definition** | Every change is deployable; deployment requires **manual approval** | Every change that passes CI is **automatically deployed** to production |
| **Human gate** | Yes (someone clicks "deploy") | No (fully automated) |
| **Risk** | Lower (human review before production) | Higher (must have excellent test coverage) |
| **Speed** | Fast (deploy on demand) | Fastest (deploy on every commit) |
| **Best for** | Most teams; regulated industries | High-maturity teams with strong test suites |

```
Continuous Delivery:
  Code → CI (build + test) → Staging (auto-deploy) → Production (manual approval → deploy)

Continuous Deployment:
  Code → CI (build + test) → Staging (auto-deploy) → Production (auto-deploy!)
```

---

### Pipeline Stages

```
Full CI/CD Pipeline:

  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  Build   │ → │  Test    │ → │  Package │ → │  Deploy  │ → │  Verify  │
  │          │   │          │   │          │   │  Staging │   │          │
  │ Compile  │   │ Unit     │   │ Docker   │   │          │   │ Smoke    │
  │ Lint     │   │ Integr.  │   │ image    │   │          │   │ tests    │
  │ Deps     │   │ E2E      │   │ Push to  │   │          │   │ Health   │
  │          │   │ Security │   │ registry │   │          │   │ checks   │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
                                                     │
                                                     ↓
                                               ┌──────────┐   ┌──────────┐
                                               │  Deploy  │ → │  Verify  │
                                               │  Prod    │   │          │
                                               │ (manual  │   │ Canary   │
                                               │  or auto)│   │ Metrics  │
                                               └──────────┘   └──────────┘
```

**CI/CD Tools:**

| Tool | Type | Key Features |
|------|------|-------------|
| **GitHub Actions** | SaaS (GitHub) | YAML workflows; tight GitHub integration; marketplace of actions |
| **GitLab CI/CD** | SaaS/Self-hosted | Built into GitLab; Auto DevOps; Kubernetes integration |
| **Jenkins** | Self-hosted | Most flexible; huge plugin ecosystem; complex to manage |
| **CircleCI** | SaaS | Fast; Docker-native; good caching |
| **AWS CodePipeline** | AWS managed | Integrates with CodeBuild, CodeDeploy, ECS, EKS |
| **ArgoCD** | Open-source | GitOps for Kubernetes; declarative; auto-sync from Git |
| **Tekton** | Open-source | Kubernetes-native CI/CD; cloud-agnostic |

---

### Deployment Strategies

**Rolling Deployment:**

Gradually replace old instances with new ones, one at a time.

```
Rolling Update (3 replicas, updating v1 → v2):

  Step 1: [v1] [v1] [v1]           ← all running v1
  Step 2: [v2] [v1] [v1]           ← 1 pod updated to v2
  Step 3: [v2] [v2] [v1]           ← 2 pods updated
  Step 4: [v2] [v2] [v2]           ← all running v2 ✅
  
  At every step, at least 2 pods are serving traffic (no downtime).
```

| Pros | Cons |
|------|------|
| Zero downtime | Both versions run simultaneously (must be backward compatible) |
| Simple (Kubernetes default) | Slow rollout for large deployments |
| Easy rollback (`kubectl rollout undo`) | Hard to test with real traffic before full rollout |

---

**Blue-Green Deployment:**

Run two identical environments (Blue = current, Green = new). Switch all traffic at once.

```
Blue-Green Deployment:

  Before:
    Load Balancer → Blue (v1) ← all traffic
                    Green (v2) ← no traffic (being tested)

  Switch:
    Load Balancer → Green (v2) ← all traffic (instant switch!)
                    Blue (v1) ← no traffic (standby for rollback)

  Rollback (if issues):
    Load Balancer → Blue (v1) ← all traffic (instant rollback!)
```

| Pros | Cons |
|------|------|
| Instant switch (zero downtime) | Requires 2x infrastructure (both environments running) |
| Instant rollback (switch back to blue) | Database migrations must be backward compatible |
| Full testing of green before switch | Higher cost (double the servers) |

---

**Canary Deployment:**

Route a small percentage of traffic to the new version. Gradually increase if metrics look good.

```
Canary Deployment:

  Step 1: 5% traffic → v2 (canary), 95% → v1 (stable)
    Monitor: error rate, latency, business metrics
    
  Step 2: 25% traffic → v2, 75% → v1
    Monitor: still looking good...
    
  Step 3: 50% traffic → v2, 50% → v1
    Monitor: all metrics healthy...
    
  Step 4: 100% traffic → v2 ✅
    v1 instances terminated

  If issues at any step:
    → Route 100% back to v1 (rollback)
    → Investigate and fix v2
```

| Pros | Cons |
|------|------|
| Low risk (only small % of users affected initially) | More complex to set up (traffic splitting) |
| Real production traffic testing | Slower rollout than blue-green |
| Data-driven rollout decisions | Requires good monitoring and metrics |
| Can catch issues that tests miss | Both versions must be backward compatible |

---

**Deployment Strategy Comparison:**

| Strategy | Downtime | Rollback Speed | Risk | Infrastructure Cost | Complexity |
|----------|---------|---------------|------|-------------------|-----------|
| **Rolling** | None | Moderate (rollback = reverse rolling) | Medium | 1x (+ 1 extra pod during update) | Low |
| **Blue-Green** | None | Instant (switch back) | Low | 2x (both environments) | Medium |
| **Canary** | None | Fast (route back to stable) | Lowest | 1x + canary instances | High |
| **Recreate** | Yes (brief) | Slow (redeploy old version) | High | 1x | Lowest |

---

### Feature Flags

**Feature flags** (feature toggles) decouple deployment from release. Deploy code to production with the feature disabled, then enable it gradually without redeploying.

```
Without Feature Flags:
  Deploy v2 with new checkout → ALL users see new checkout immediately
  Bug found → must rollback entire deployment

With Feature Flags:
  Deploy v2 with new checkout (flag: OFF) → no users see it
  Enable flag for 5% of users → test with real traffic
  Enable flag for 50% → broader testing
  Enable flag for 100% → full release
  Bug found → disable flag instantly (no deployment needed!)
```

**Feature Flag Use Cases:**

| Use Case | Description |
|----------|-------------|
| **Gradual rollout** | Enable for 5% → 25% → 50% → 100% of users |
| **A/B testing** | Show variant A to 50%, variant B to 50%; measure which performs better |
| **Kill switch** | Instantly disable a feature if it causes problems |
| **Beta testing** | Enable for specific users (beta testers, internal employees) |
| **Trunk-based development** | Merge incomplete features to main behind a flag; enable when ready |
| **Operational toggles** | Disable expensive features during high load (graceful degradation) |

**Feature Flag Tools:**

| Tool | Type | Key Features |
|------|------|-------------|
| **LaunchDarkly** | SaaS | Industry leader; targeting rules; A/B testing; SDKs for 25+ languages |
| **Unleash** | Open-source | Self-hosted; gradual rollout; A/B testing |
| **Flagsmith** | Open-source/SaaS | Feature flags + remote config |
| **AWS AppConfig** | AWS managed | Feature flags + configuration management |
| **Split.io** | SaaS | Feature flags + experimentation platform |

---


## 31.4 Infrastructure as Code

> **Infrastructure as Code (IaC)** manages infrastructure (servers, networks, databases, load balancers) through **code and configuration files** rather than manual processes. IaC enables version control, code review, automated testing, and reproducible environments for infrastructure — the same practices used for application code.

---

### Terraform

**Terraform** (HashiCorp) is the most popular IaC tool. It uses a declarative language (HCL) to define infrastructure and supports all major cloud providers.

```hcl
# main.tf — Define a VPC, subnet, and EC2 instance on AWS

provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "main-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "public-subnet" }
}

resource "aws_instance" "api_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  subnet_id     = aws_subnet.public.id
  
  tags = { Name = "api-server", Environment = "production" }
}

resource "aws_db_instance" "postgres" {
  engine               = "postgres"
  engine_version       = "16.1"
  instance_class       = "db.r6g.large"
  allocated_storage    = 100
  db_name              = "mydb"
  username             = "admin"
  password             = var.db_password  # from variable (never hardcode!)
  skip_final_snapshot  = false
  multi_az             = true             # high availability
}
```

**Terraform Workflow:**

```
1. terraform init     → download provider plugins
2. terraform plan     → preview changes (what will be created/modified/destroyed)
3. terraform apply    → apply changes (create/modify infrastructure)
4. terraform destroy  → tear down all infrastructure

State:
  Terraform stores the current state of infrastructure in a state file (terraform.tfstate).
  In production, store state remotely (S3 + DynamoDB for locking).
```

**Terraform Key Concepts:**

| Concept | Description |
|---------|-------------|
| **Provider** | Plugin for a cloud/service (AWS, GCP, Azure, Kubernetes, Cloudflare) |
| **Resource** | A piece of infrastructure (EC2 instance, S3 bucket, RDS database) |
| **Data Source** | Read existing infrastructure (look up an AMI, VPC, etc.) |
| **Variable** | Input parameters (instance type, region, environment) |
| **Output** | Export values (IP addresses, DNS names) for use by other configs |
| **Module** | Reusable group of resources (VPC module, EKS module) |
| **State** | Tracks what infrastructure exists; stored remotely for team collaboration |
| **Plan** | Preview of changes before applying (safe — no changes made) |

---

### CloudFormation

**AWS CloudFormation** is AWS's native IaC service. It uses YAML or JSON templates to define AWS resources.

```yaml
# cloudformation-template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: API Server Stack

Resources:
  ApiInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.medium
      ImageId: ami-0c55b159cbfafe1f0
      SubnetId: !Ref PublicSubnet
      Tags:
        - Key: Name
          Value: api-server

  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: postgres
      EngineVersion: '16.1'
      DBInstanceClass: db.r6g.large
      AllocatedStorage: 100
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      MultiAZ: true
```

**Terraform vs CloudFormation:**

| Feature | Terraform | CloudFormation |
|---------|-----------|---------------|
| **Cloud support** | Multi-cloud (AWS, GCP, Azure, 1000+ providers) | AWS only |
| **Language** | HCL (HashiCorp Configuration Language) | YAML or JSON |
| **State management** | External (S3 + DynamoDB) | Managed by AWS (automatic) |
| **Drift detection** | `terraform plan` (manual) | Built-in drift detection |
| **Rollback** | Manual (apply previous state) | Automatic rollback on failure |
| **Ecosystem** | Terraform Registry (modules) | AWS-specific |
| **Learning curve** | Moderate | Moderate (AWS-specific knowledge needed) |
| **Best for** | Multi-cloud, hybrid | AWS-only environments |

---

### Ansible

**Ansible** (Red Hat) is a configuration management and automation tool. Unlike Terraform (which creates infrastructure), Ansible **configures** existing infrastructure (install software, configure services, deploy applications).

```yaml
# playbook.yml — Configure a web server
- hosts: web_servers
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Copy Nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

**Terraform vs Ansible:**

| Feature | Terraform | Ansible |
|---------|-----------|---------|
| **Purpose** | Provision infrastructure (create VMs, networks, databases) | Configure infrastructure (install software, deploy apps) |
| **Approach** | Declarative ("I want 3 servers") | Procedural + Declarative ("install nginx, copy config") |
| **State** | Maintains state file | Stateless (runs tasks each time) |
| **Idempotent** | Yes (declarative) | Yes (most modules are idempotent) |
| **Agent** | Agentless (API calls to cloud providers) | Agentless (SSH to servers) |
| **Best for** | Creating cloud infrastructure | Configuring servers, deploying applications |

**Common Pattern:** Use **Terraform** to create infrastructure (VMs, networks, databases) and **Ansible** to configure it (install software, deploy applications).

---

### Immutable Infrastructure

**Mutable Infrastructure (traditional):**
Servers are updated in place — install patches, update configurations, deploy new code on existing servers. Over time, servers "drift" from their intended state (snowflake servers).

**Immutable Infrastructure (modern):**
Servers are **never modified** after creation. To update, build a **new image** with the changes and replace the old servers entirely.

```
Mutable (traditional):
  Server created → patch OS → install app v1 → update config → install app v2 → patch OS again
  (Server accumulates changes over months/years → "snowflake" — unique, hard to reproduce)

Immutable (modern):
  Build image with app v1 → deploy 3 servers from image
  Build NEW image with app v2 → deploy 3 NEW servers → terminate old servers
  (Every server is identical, created from the same image, never modified)
```

| Feature | Mutable | Immutable |
|---------|---------|-----------|
| **Updates** | Modify existing servers | Replace with new servers |
| **Consistency** | Drift over time (snowflakes) | Always consistent (from image) |
| **Rollback** | Complex (undo changes) | Simple (deploy previous image) |
| **Debugging** | Hard (each server may be different) | Easy (all servers are identical) |
| **Security** | Patches applied incrementally | Fresh image with all patches |
| **Tools** | Ansible, Chef, Puppet | Docker, Packer, AMI, Kubernetes |

**Key Point:** Containers (Docker) and Kubernetes naturally enforce immutable infrastructure — you don't SSH into a container to update it; you build a new image and deploy it.

---


## 31.5 Cloud Services (Overview)

> Modern system design relies heavily on cloud services. Instead of managing your own servers, databases, and networking, you use managed services from AWS, GCP, or Azure. Understanding the key services and when to use each is essential for system design interviews.

---

### AWS Services Map

```
AWS Services for System Design:

  ┌─────────────────────────────────────────────────────────────────┐
  │                        COMPUTE                                  │
  │  EC2 (VMs)  │  Lambda (Serverless)  │  ECS/EKS (Containers)    │
  ├─────────────────────────────────────────────────────────────────┤
  │                        STORAGE                                  │
  │  S3 (Objects)  │  EBS (Block)  │  EFS (File)  │  Glacier       │
  ├─────────────────────────────────────────────────────────────────┤
  │                        DATABASE                                 │
  │  RDS (SQL)  │  Aurora  │  DynamoDB (NoSQL)  │  ElastiCache     │
  ├─────────────────────────────────────────────────────────────────┤
  │                        NETWORKING                               │
  │  VPC  │  Route 53 (DNS)  │  CloudFront (CDN)  │  ELB (LB)     │
  ├─────────────────────────────────────────────────────────────────┤
  │                        MESSAGING                                │
  │  SQS (Queue)  │  SNS (Pub/Sub)  │  EventBridge  │  Kinesis     │
  ├─────────────────────────────────────────────────────────────────┤
  │                        SECURITY                                 │
  │  IAM  │  KMS  │  Secrets Manager  │  WAF  │  Shield           │
  ├─────────────────────────────────────────────────────────────────┤
  │                        OBSERVABILITY                            │
  │  CloudWatch  │  X-Ray  │  CloudTrail                           │
  └─────────────────────────────────────────────────────────────────┘
```

---

### Compute

| Service | Type | Use Case | Scaling |
|---------|------|----------|---------|
| **EC2** | Virtual machines | Full control over OS and runtime; legacy apps; custom software | Auto Scaling Groups |
| **Lambda** | Serverless functions | Event-driven; short-lived tasks (< 15 min); API handlers | Automatic (per-request) |
| **ECS** | Container orchestration (AWS-native) | Docker containers without Kubernetes complexity | Service auto-scaling |
| **EKS** | Managed Kubernetes | Kubernetes workloads; multi-cloud portability | HPA, Cluster Autoscaler |
| **Fargate** | Serverless containers | Containers without managing servers (works with ECS/EKS) | Automatic |

**When to Use Which:**

| Scenario | Service |
|----------|---------|
| Full OS control, custom AMIs | EC2 |
| Simple event-driven functions (API, S3 trigger, cron) | Lambda |
| Docker containers, AWS-native | ECS + Fargate |
| Kubernetes, multi-cloud, complex orchestration | EKS |
| Batch processing, ML training | EC2 (GPU instances) or SageMaker |

**Lambda vs Containers:**

| Feature | Lambda | Containers (ECS/EKS) |
|---------|--------|---------------------|
| Max execution time | 15 minutes | Unlimited |
| Cold start | 100ms - 10s (depends on runtime) | None (always running) |
| Scaling | Instant (per-request) | Seconds-minutes (add containers) |
| Cost model | Pay per invocation + duration | Pay for running containers |
| State | Stateless (no local state between invocations) | Can be stateful |
| Best for | Event-driven, sporadic traffic, simple functions | Long-running services, complex apps, steady traffic |

---

### Storage

| Service | Type | Durability | Use Case |
|---------|------|-----------|----------|
| **S3** | Object storage | 11 nines | Static assets, backups, data lake, media files |
| **EBS** | Block storage | 99.999% | Database volumes, OS boot volumes |
| **EFS** | File storage (NFS) | 11 nines | Shared file system across EC2/ECS instances |
| **S3 Glacier** | Archive storage | 11 nines | Long-term backups, compliance archives |
| **FSx** | Managed file systems | High | Lustre (HPC), Windows File Server, NetApp |

---

### Database

| Service | Type | Engine | Use Case |
|---------|------|--------|----------|
| **RDS** | Managed relational DB | PostgreSQL, MySQL, MariaDB, Oracle, SQL Server | Traditional SQL workloads |
| **Aurora** | AWS-optimized relational | MySQL-compatible, PostgreSQL-compatible | High-performance SQL; 5x MySQL throughput |
| **DynamoDB** | Managed NoSQL (key-value + document) | Proprietary | High-scale key-value; serverless; single-digit ms latency |
| **ElastiCache** | Managed cache | Redis, Memcached | Caching, session storage, leaderboards |
| **DocumentDB** | Managed document DB | MongoDB-compatible | MongoDB workloads on AWS |
| **Neptune** | Managed graph DB | Property graph, RDF | Social networks, recommendations, fraud detection |
| **Redshift** | Data warehouse | Columnar (PostgreSQL-based) | Analytics, BI, OLAP |
| **Keyspaces** | Managed Cassandra | Cassandra-compatible | Wide-column workloads |
| **MemoryDB** | Durable Redis | Redis-compatible | Redis with durability (primary database, not just cache) |

---

### Networking

| Service | Purpose | Use Case |
|---------|---------|----------|
| **VPC** | Isolated virtual network | Network isolation, subnets, security groups |
| **Route 53** | DNS + health checks | Domain management, DNS failover, GeoDNS |
| **CloudFront** | CDN | Static asset delivery, API caching, edge functions (Lambda@Edge) |
| **ALB** | L7 load balancer | HTTP/HTTPS routing, path-based routing, WebSocket |
| **NLB** | L4 load balancer | TCP/UDP, ultra-low latency, static IPs |
| **API Gateway** | Managed API gateway | REST/WebSocket APIs, Lambda integration, throttling |
| **PrivateLink** | Private connectivity | Access AWS services without going through the internet |
| **Transit Gateway** | Network hub | Connect multiple VPCs and on-premises networks |

---

### Messaging

| Service | Type | Use Case |
|---------|------|----------|
| **SQS** | Message queue (point-to-point) | Decoupling services, task queues, buffering |
| **SNS** | Pub/Sub (fan-out) | Notifications, event broadcasting to multiple subscribers |
| **EventBridge** | Event bus | Event-driven architectures, SaaS integrations, scheduled events |
| **Kinesis** | Real-time streaming | Real-time analytics, log aggregation, IoT data streams |
| **MSK** | Managed Kafka | Kafka workloads on AWS |
| **MQ** | Managed message broker | ActiveMQ/RabbitMQ workloads (legacy migration) |

**SQS vs SNS vs EventBridge:**

| Feature | SQS | SNS | EventBridge |
|---------|-----|-----|-------------|
| Pattern | Queue (point-to-point) | Pub/Sub (fan-out) | Event bus (routing) |
| Consumers | One consumer per message | Multiple subscribers | Multiple targets with filtering |
| Ordering | FIFO (optional) | No ordering guarantee | No ordering guarantee |
| Filtering | No (consumer gets all messages) | Attribute-based filtering | Content-based filtering (powerful) |
| Retry | Built-in (visibility timeout) | Retry policies per subscription | Built-in retry + DLQ |
| Best for | Task queues, decoupling | Notifications, fan-out | Event-driven architectures, complex routing |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Containers | Package app + dependencies; consistent across environments; Docker is the standard |
| Container vs VM | Containers: lightweight (MBs), fast startup (seconds); VMs: stronger isolation (GBs, minutes) |
| Dockerfile | Multi-stage builds for small images; don't run as root; use specific tags; order by change frequency |
| Docker Compose | Multi-container local development; YAML definition; depends_on for ordering |
| Kubernetes | Container orchestration; Pods, Deployments, Services, Ingress; auto-scaling, self-healing |
| K8s Deployment | Manages ReplicaSet; rolling updates; rollback with `kubectl rollout undo` |
| K8s Service | Stable endpoint for pods; ClusterIP (internal), LoadBalancer (external) |
| ConfigMaps/Secrets | Configuration and sensitive data; use External Secrets Operator for production |
| Ingress | HTTP routing rules; path-based routing to different services; TLS termination |
| CI | Automated build + test on every push; fast feedback (< 10 min); fail fast |
| CD | Continuous Delivery (manual deploy gate) vs Continuous Deployment (auto deploy) |
| Rolling Deployment | Gradual replacement; zero downtime; Kubernetes default |
| Blue-Green | Two environments; instant switch; instant rollback; 2x infrastructure cost |
| Canary | Small % of traffic to new version; data-driven rollout; lowest risk |
| Feature Flags | Decouple deployment from release; gradual rollout; instant kill switch |
| Terraform | Multi-cloud IaC; declarative (HCL); plan before apply; state management |
| CloudFormation | AWS-native IaC; YAML/JSON; automatic rollback on failure |
| Ansible | Configuration management; install software, deploy apps; agentless (SSH) |
| Immutable Infrastructure | Never modify servers; build new image, replace old servers; containers enforce this |
| AWS Compute | EC2 (VMs), Lambda (serverless), ECS/EKS (containers), Fargate (serverless containers) |
| AWS Database | RDS/Aurora (SQL), DynamoDB (NoSQL), ElastiCache (Redis), Redshift (warehouse) |
| AWS Messaging | SQS (queue), SNS (pub/sub), EventBridge (event bus), Kinesis (streaming) |

---

## Interview Tips for Module 31

1. **Containers** — explain the difference between containers and VMs; mention Docker multi-stage builds for small images
2. **Kubernetes** — draw the architecture (control plane + worker nodes); explain Pod, Deployment, Service, Ingress
3. **K8s Deployment** — explain rolling update strategy; mention maxSurge and maxUnavailable
4. **CI/CD pipeline** — draw the stages (build → test → package → deploy staging → deploy prod); mention the tools
5. **Deployment strategies** — explain rolling, blue-green, canary with trade-offs; canary is the safest for production
6. **Feature flags** — explain how they decouple deployment from release; mention gradual rollout and kill switch
7. **Terraform** — explain declarative IaC; plan → apply workflow; mention state management
8. **Terraform vs CloudFormation** — Terraform is multi-cloud; CloudFormation is AWS-only with automatic rollback
9. **Immutable infrastructure** — explain why it's better than mutable; containers naturally enforce it
10. **AWS services** — know the key services per category; be ready to pick the right service for a given requirement
11. **Lambda vs Containers** — Lambda for event-driven/sporadic; containers for long-running/steady traffic
12. **SQS vs SNS vs EventBridge** — SQS for queues, SNS for fan-out, EventBridge for event routing with filtering
