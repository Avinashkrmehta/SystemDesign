# Module 31: DevOps & Deployment

> DevOps bridges the gap between development and operations — enabling teams to build, test, and deploy software faster and more reliably. In system design, understanding containerization, orchestration, CI/CD pipelines, deployment strategies, infrastructure as code, and cloud services is essential for designing systems that can be operated at scale. This module covers the tools and practices that turn code into running, reliable production systems.

> **Ruby Context:** The Ruby/Rails deployment ecosystem has evolved significantly. Key tools include **Docker** (containerizing Rails + Puma + Sidekiq), **Kubernetes** (orchestrating Rails pods), **Kamal** (Rails-native deployment tool from 37signals, formerly MRSK), **Capistrano** (traditional SSH-based deployment), **GitHub Actions** (CI/CD with Ruby-specific actions), **Terraform** (infrastructure provisioning), and **Heroku** / **Render** / **Fly.io** (PaaS platforms). Rails 7.1+ includes a production-ready Dockerfile generator (`bin/rails generate dockerfile`), and Kamal is bundled with Rails 8+ as the default deployment tool.

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
|--------|--------|---------|        |  Rails App  |  Sidekiq   |
| Libs A | Libs B | Libs C  |        |  Puma       |  Worker    |
|--------|--------|---------|        |  Ruby 3.3   |  Ruby 3.3  |
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

> **Ruby Context:** Rails 7.1+ includes `bin/rails generate dockerfile` which creates a production-optimized Dockerfile. Here's a best-practice Dockerfile for a Rails application.

```dockerfile
# Production Dockerfile for Rails (multi-stage build)

# Stage 1: Build gems and assets
FROM ruby:3.3-slim AS builder

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    build-essential libpq-dev git node-modules && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install gems (cached layer — only re-runs when Gemfile changes)
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local deployment true && \
    bundle config set --local without 'development test' && \
    bundle install --jobs 4 --retry 3 && \
    rm -rf ~/.bundle/cache

# Install JavaScript dependencies (if using jsbundling/cssbundling)
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile --production

# Copy application code
COPY . .

# Precompile assets
RUN SECRET_KEY_BASE=placeholder bundle exec rails assets:precompile

# Remove unnecessary files
RUN rm -rf node_modules tmp/cache vendor/bundle/ruby/*/cache


# Stage 2: Runtime (minimal image)
FROM ruby:3.3-slim

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Create non-root user
RUN groupadd --system app && useradd --system --gid app app

# Copy built artifacts from builder stage
COPY --from=builder /app /app
COPY --from=builder /usr/local/bundle /usr/local/bundle

# Set environment
ENV RAILS_ENV=production \
    RAILS_LOG_TO_STDOUT=true \
    RAILS_SERVE_STATIC_FILES=true \
    BUNDLE_DEPLOYMENT=true \
    BUNDLE_WITHOUT="development:test"

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/up || exit 1

# Don't run as root
USER app

EXPOSE 3000

# Start Puma
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

**Dockerfile Best Practices:**

| Practice | Why |
|----------|-----|
| **Use multi-stage builds** | Build stage has compilers/node; runtime stage is slim (~150 MB vs ~800 MB) |
| **Use specific base image tags** | `ruby:3.3-slim` not `ruby:latest` — reproducible builds |
| **Copy Gemfile first** | Layer caching — gems only re-install when Gemfile changes |
| **Don't run as root** | Use `USER app` — limits damage if container is compromised |
| **Use .dockerignore** | Exclude `.git`, `tmp`, `log`, `node_modules`, `spec/`, `.env` |
| **Use slim base images** | `ruby:3.3-slim` (~80 MB) instead of `ruby:3.3` (~900 MB) |
| **Precompile assets in build** | Assets are compiled once during build, not on every container start |
| **Set RAILS_LOG_TO_STDOUT** | Logs go to stdout for container log collection (Fluentd/Filebeat) |

```
# .dockerignore
.git
.github
tmp
log
node_modules
spec
test
.env*
.rspec
coverage
```

---

### Docker Compose

**Docker Compose** defines and runs multi-container applications. Ideal for local Rails development.

> **Ruby Context:** A typical Rails development setup includes the Rails app (Puma), Sidekiq workers, PostgreSQL, Redis, and optionally Elasticsearch or Kafka.

```yaml
# docker-compose.yml for Rails development
version: '3.8'

services:
  web:
    build: .
    command: bundle exec puma -C config/puma.rb
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp_development
      - REDIS_URL=redis://redis:6379/0
      - RAILS_ENV=development
    volumes:
      - .:/app                    # Mount source code for live reload
      - bundle_cache:/usr/local/bundle  # Cache gems across rebuilds
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/up"]
      interval: 10s
      timeout: 5s
      retries: 3

  sidekiq:
    build: .
    command: bundle exec sidekiq -C config/sidekiq.yml
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp_development
      - REDIS_URL=redis://redis:6379/0
      - RAILS_ENV=development
    volumes:
      - .:/app
      - bundle_cache:/usr/local/bundle
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp_development
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
  bundle_cache:
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
  Build image: ruby:3.3 (900 MB) + gems + node_modules + build tools
  → Final image: 1.2+ GB

With Multi-Stage:
  Stage 1 (builder): ruby:3.3 (900 MB) → install gems, compile assets
  Stage 2 (runtime): ruby:3.3-slim (80 MB) + gems + compiled assets
  → Final image: ~200 MB (83% smaller!)
```

**Multi-Stage for Different Languages:**

| Language | Builder Image | Runtime Image | Typical Final Size |
|----------|-------------|--------------|-------------------|
| **Ruby/Rails** | ruby:3.3 | ruby:3.3-slim | 150-300 MB |
| Go | golang:1.22 | alpine or scratch | 10-30 MB |
| Java | maven:3.9 or gradle:8 | eclipse-temurin:21-jre-alpine | 150-300 MB |
| Node.js | node:20 | node:20-alpine | 100-200 MB |
| Python | python:3.12 | python:3.12-slim | 100-200 MB |

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
  |  | Rails |  | Rails |    |  |  | Rails |  |Sidekiq|    |
  |  | Pod 1 |  | Pod 2 |    |  |  | Pod 3 |  | Pod 1 |    |
  |  +-------+  +-------+    |  |  +-------+  +-------+    |
  |  +---------------------+  |  |  +---------------------+  |
  |  | kubelet | kube-proxy|  |  |  | kubelet | kube-proxy|  |
  |  +---------------------+  |  |  +---------------------+  |
  +---------------------------+  +---------------------------+
```

---

### Core Kubernetes Objects

> **Ruby Context:** A typical Rails deployment on Kubernetes includes separate Deployments for Puma (web) and Sidekiq (worker), Services for routing, and ConfigMaps/Secrets for configuration.

**Deployment (Rails Web — Puma):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-web
  labels:
    app: myapp
    component: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      component: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
        component: web
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: rails
          image: myapp/web:1.2.3-abc123
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: rails-config
            - secretRef:
                name: rails-secrets
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          startupProbe:
            httpGet:
              path: /up
              port: 3000
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /up
              port: 3000
            periodSeconds: 10
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            periodSeconds: 5
            timeoutSeconds: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
```

**Deployment (Sidekiq Worker):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-sidekiq
  labels:
    app: myapp
    component: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      component: worker
  template:
    metadata:
      labels:
        app: myapp
        component: worker
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: sidekiq
          image: myapp/web:1.2.3-abc123    # Same image as web!
          command: ["bundle", "exec", "sidekiq", "-C", "config/sidekiq.yml"]
          envFrom:
            - configMapRef:
                name: rails-config
            - secretRef:
                name: rails-secrets
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          # No HTTP probes for Sidekiq — use exec probe instead
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - "bundle exec sidekiqmon processes | grep -q 'busy'"
            periodSeconds: 30
            timeoutSeconds: 5
```

**Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rails-web
spec:
  selector:
    app: myapp
    component: web
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
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

```yaml
# ConfigMap for Rails
apiVersion: v1
kind: ConfigMap
metadata:
  name: rails-config
data:
  RAILS_ENV: "production"
  RAILS_LOG_TO_STDOUT: "true"
  RAILS_SERVE_STATIC_FILES: "true"
  REDIS_URL: "redis://redis-master:6379/0"
  SIDEKIQ_CONCURRENCY: "10"
  PUMA_WORKERS: "2"
  PUMA_THREADS: "5"

---
# Secret for Rails (use External Secrets Operator in production)
apiVersion: v1
kind: Secret
metadata:
  name: rails-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@rds-endpoint:5432/myapp_production"
  SECRET_KEY_BASE: "a1b2c3d4e5f6..."
  STRIPE_SECRET_KEY: "sk_live_..."
```

**External Secrets Management:**
For production, use an external secrets manager instead of Kubernetes Secrets:
- **External Secrets Operator** — syncs secrets from Vault/AWS Secrets Manager into K8s Secrets
- **Sealed Secrets** — encrypts secrets so they can be safely stored in Git
- **HashiCorp Vault** — inject secrets directly into pods via sidecar or CSI driver

---

### Ingress Controllers

```yaml
# Ingress for Rails application
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rails-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"  # For file uploads
spec:
  tls:
    - hosts:
        - myapp.com
        - api.myapp.com
      secretName: myapp-tls
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rails-web
                port:
                  number: 80
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rails-web
                port:
                  number: 80
```

---

### Database Migrations in Kubernetes

> **Ruby Context:** Running `rails db:migrate` in Kubernetes requires special handling — you don't want every pod running migrations simultaneously. Use a Kubernetes Job or an init container.

```yaml
# Kubernetes Job for Rails migrations (run before deployment)
apiVersion: batch/v1
kind: Job
metadata:
  name: rails-migrate-1-2-3
  annotations:
    argocd.argoproj.io/hook: PreSync  # Run before deployment (ArgoCD)
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp/web:1.2.3-abc123
          command: ["bundle", "exec", "rails", "db:migrate"]
          envFrom:
            - configMapRef:
                name: rails-config
            - secretRef:
                name: rails-secrets
```

---


## 31.3 CI/CD

> **CI/CD** (Continuous Integration / Continuous Delivery / Continuous Deployment) automates the process of building, testing, and deploying software. It's the backbone of modern software delivery — enabling teams to ship changes multiple times per day with confidence.

---

### Continuous Integration (CI)

**CI** is the practice of frequently merging code changes into a shared repository, with each merge triggering an automated build and test pipeline.

> **Ruby Context:** A typical Rails CI pipeline runs RuboCop (linting), RSpec/Minitest (tests), Brakeman (security scanning), and bundler-audit (dependency vulnerabilities), then builds a Docker image.

```yaml
# .github/workflows/ci.yml — GitHub Actions CI for Rails
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - run: bundle exec rubocop --parallel
      - run: bundle exec brakeman --no-pager
      - run: bundle exec bundler-audit check --update

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: myapp_test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']

    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Setup database
        env:
          DATABASE_URL: postgres://postgres:password@localhost:5432/myapp_test
          RAILS_ENV: test
        run: bundle exec rails db:schema:load

      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:password@localhost:5432/myapp_test
          REDIS_URL: redis://localhost:6379/0
          RAILS_ENV: test
        run: bundle exec rspec --format progress --format RspecJunitFormatter --out tmp/rspec.xml

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: tmp/rspec.xml

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            myapp/web:${{ github.sha }}
            myapp/web:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**CI Best Practices:**

| Practice | Why |
|----------|-----|
| **Run on every push/PR** | Catch issues early, before they reach main branch |
| **Fast feedback** (< 10 minutes) | Developers wait for results; slow CI → slow development |
| **Fail fast** | Run linting first (fast); tests later (slow); security scan in parallel |
| **Cache gems** | `bundler-cache: true` in `ruby/setup-ruby` caches gems across runs |
| **Parallel tests** | Use `parallel_tests` gem or split RSpec across CI workers |
| **Branch protection** | Require CI to pass before merging to main |

---

### Continuous Delivery vs Continuous Deployment

| Feature | Continuous Delivery | Continuous Deployment |
|---------|-------------------|---------------------|
| **Definition** | Every change is deployable; deployment requires **manual approval** | Every change that passes CI is **automatically deployed** to production |
| **Human gate** | Yes (someone clicks "deploy") | No (fully automated) |
| **Risk** | Lower (human review before production) | Higher (must have excellent test coverage) |
| **Speed** | Fast (deploy on demand) | Fastest (deploy on every commit) |
| **Best for** | Most teams; regulated industries | High-maturity teams with strong test suites |

---

### Deployment Strategies

**Rolling Deployment:**

```
Rolling Update (3 replicas, updating v1 → v2):

  Step 1: [v1] [v1] [v1]           ← all running v1
  Step 2: [v2] [v1] [v1]           ← 1 pod updated to v2
  Step 3: [v2] [v2] [v1]           ← 2 pods updated
  Step 4: [v2] [v2] [v2]           ← all running v2 ✅
```

| Pros | Cons |
|------|------|
| Zero downtime | Both versions run simultaneously (must be backward compatible) |
| Simple (Kubernetes default) | Slow rollout for large deployments |
| Easy rollback (`kubectl rollout undo`) | Hard to test with real traffic before full rollout |

---

**Blue-Green Deployment:**

```
  Before:
    Load Balancer → Blue (v1) ← all traffic
                    Green (v2) ← no traffic (being tested)

  Switch:
    Load Balancer → Green (v2) ← all traffic (instant switch!)
                    Blue (v1) ← no traffic (standby for rollback)
```

| Pros | Cons |
|------|------|
| Instant switch (zero downtime) | Requires 2x infrastructure |
| Instant rollback | Database migrations must be backward compatible |
| Full testing of green before switch | Higher cost |

---

**Canary Deployment:**

```
  Step 1: 5% traffic → v2 (canary), 95% → v1 (stable)
  Step 2: 25% traffic → v2, 75% → v1
  Step 3: 50% traffic → v2, 50% → v1
  Step 4: 100% traffic → v2 ✅

  If issues at any step → route 100% back to v1
```

| Pros | Cons |
|------|------|
| Low risk (only small % affected initially) | More complex to set up |
| Real production traffic testing | Slower rollout |
| Data-driven rollout decisions | Requires good monitoring |

---

**Deployment Strategy Comparison:**

| Strategy | Downtime | Rollback Speed | Risk | Infrastructure Cost | Complexity |
|----------|---------|---------------|------|-------------------|-----------|
| **Rolling** | None | Moderate | Medium | 1x + 1 extra pod | Low |
| **Blue-Green** | None | Instant | Low | 2x | Medium |
| **Canary** | None | Fast | Lowest | 1x + canary | High |
| **Recreate** | Yes (brief) | Slow | High | 1x | Lowest |

---

### Kamal (Rails-Native Deployment)

> **Ruby Context:** **Kamal** (formerly MRSK, from 37signals/Basecamp) is a deployment tool built specifically for Rails. It deploys Docker containers to bare servers via SSH — no Kubernetes needed. It's bundled with Rails 8+ as the default deployment tool.

```ruby
# config/deploy.yml — Kamal configuration
service: myapp

image: myapp/web

servers:
  web:
    hosts:
      - 192.168.1.10
      - 192.168.1.11
    labels:
      role: web
    options:
      memory: 1g
  worker:
    hosts:
      - 192.168.1.12
    cmd: bundle exec sidekiq -C config/sidekiq.yml
    labels:
      role: worker

registry:
  server: ghcr.io
  username: myorg
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
    RAILS_LOG_TO_STDOUT: true
  secret:
    - DATABASE_URL
    - REDIS_URL
    - SECRET_KEY_BASE

traefik:
  options:
    publish:
      - "443:443"
    volume:
      - "/letsencrypt:/letsencrypt"

healthcheck:
  path: /up
  port: 3000
  interval: 10s

# Deploy commands:
# kamal setup          — first-time setup (install Docker, Traefik, deploy)
# kamal deploy         — build, push, deploy new version
# kamal rollback       — rollback to previous version
# kamal app logs       — view application logs
# kamal app exec 'rails console' — run Rails console on a server
```

**Kamal vs Kubernetes vs Capistrano:**

| Feature | Kamal | Kubernetes | Capistrano |
|---------|-------|-----------|-----------|
| **Complexity** | Low | High | Low |
| **Infrastructure** | Bare servers (SSH) | K8s cluster | Bare servers (SSH) |
| **Container support** | Docker (required) | Docker/containerd | No (deploys code directly) |
| **Auto-scaling** | No (manual) | Yes (HPA) | No |
| **Load balancing** | Traefik (built-in) | Ingress controllers | External (Nginx) |
| **Zero-downtime** | Yes (rolling via Traefik) | Yes (rolling update) | Yes (with config) |
| **Best for** | Small-medium Rails apps | Large-scale, multi-service | Legacy Rails apps |
| **Learning curve** | Gentle | Steep | Gentle |

---

### Feature Flags

**Feature flags** decouple deployment from release. Deploy code to production with the feature disabled, then enable it gradually.

> **Ruby Context:** **Flipper** is the most popular feature flag gem for Rails. It supports percentage rollouts, actor-based targeting, and multiple storage backends (Redis, ActiveRecord, memory).

```ruby
# Gemfile: gem 'flipper'
#          gem 'flipper-redis'  (or flipper-active_record)
#          gem 'flipper-ui'     (web UI for managing flags)

# config/initializers/flipper.rb
Flipper.configure do |config|
  config.adapter { Flipper::Adapters::Redis.new(Redis.new(url: ENV['REDIS_URL'])) }
end

# config/routes.rb
mount Flipper::UI.app(Flipper) => '/admin/flipper'  # Web UI

# Usage in application code
class CheckoutController < ApplicationController
  def create
    if Flipper.enabled?(:new_checkout, current_user)
      # New checkout flow
      NewCheckoutService.call(current_user, cart)
    else
      # Old checkout flow
      LegacyCheckoutService.call(current_user, cart)
    end
  end
end

# Enable for specific users (beta testers)
Flipper.enable(:new_checkout, User.find(1))
Flipper.enable(:new_checkout, User.find(2))

# Enable for a percentage of users (gradual rollout)
Flipper.enable_percentage_of_actors(:new_checkout, 10)  # 10% of users

# Enable for everyone
Flipper.enable(:new_checkout)

# Disable instantly (kill switch)
Flipper.disable(:new_checkout)

# Group-based targeting
Flipper.register(:staff) { |actor| actor.respond_to?(:staff?) && actor.staff? }
Flipper.enable_group(:new_checkout, :staff)  # Enable for all staff
```

**Feature Flag Tools:**

| Tool | Type | Key Features |
|------|------|-------------|
| **Flipper** | Ruby gem | Rails-native; percentage rollout; actor targeting; web UI |
| **LaunchDarkly** | SaaS | Industry leader; targeting rules; A/B testing; SDKs for 25+ languages |
| **Unleash** | Open-source | Self-hosted; gradual rollout; A/B testing |
| **Flagsmith** | Open-source/SaaS | Feature flags + remote config |
| **AWS AppConfig** | AWS managed | Feature flags + configuration management |

---


## 31.4 Infrastructure as Code

> **Infrastructure as Code (IaC)** manages infrastructure (servers, networks, databases, load balancers) through **code and configuration files** rather than manual processes. IaC enables version control, code review, automated testing, and reproducible environments for infrastructure.

---

### Terraform

**Terraform** (HashiCorp) is the most popular IaC tool. It uses a declarative language (HCL) to define infrastructure and supports all major cloud providers.

> **Ruby Context:** Terraform is commonly used to provision the infrastructure that Rails apps run on — VPCs, RDS instances, ElastiCache, S3 buckets, ECS/EKS clusters, and ALBs.

```hcl
# main.tf — Infrastructure for a Rails application on AWS

provider "aws" {
  region = "us-east-1"
}

# RDS PostgreSQL for Rails
resource "aws_db_instance" "rails_db" {
  engine               = "postgres"
  engine_version       = "16.1"
  instance_class       = "db.r6g.large"
  allocated_storage    = 100
  db_name              = "myapp_production"
  username             = "rails"
  password             = var.db_password
  skip_final_snapshot  = false
  multi_az             = true
  storage_encrypted    = true

  vpc_security_group_ids = [aws_security_group.database.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  tags = { Name = "rails-production-db", Environment = "production" }
}

# ElastiCache Redis for Rails (cache + Sidekiq)
resource "aws_elasticache_replication_group" "rails_redis" {
  replication_group_id = "rails-redis"
  description          = "Redis for Rails cache and Sidekiq"
  engine               = "redis"
  engine_version       = "7.0"
  node_type            = "cache.r6g.large"
  num_cache_clusters   = 2
  automatic_failover_enabled = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true

  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.redis.id]
}

# S3 bucket for ActiveStorage uploads
resource "aws_s3_bucket" "uploads" {
  bucket = "myapp-uploads-production"
  tags   = { Name = "rails-uploads", Environment = "production" }
}

resource "aws_s3_bucket_versioning" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  rule {
    id     = "transition-to-ia"
    status = "Enabled"
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
  }
}

# Output values for Rails configuration
output "database_url" {
  value     = "postgres://${aws_db_instance.rails_db.username}:${var.db_password}@${aws_db_instance.rails_db.endpoint}/${aws_db_instance.rails_db.db_name}"
  sensitive = true
}

output "redis_url" {
  value = "rediss://${aws_elasticache_replication_group.rails_redis.primary_endpoint_address}:6379"
}

output "s3_bucket" {
  value = aws_s3_bucket.uploads.bucket
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
| **Variable** | Input parameters (instance type, region, environment) |
| **Output** | Export values (database URL, Redis URL) for use by Rails configuration |
| **Module** | Reusable group of resources (VPC module, EKS module) |
| **State** | Tracks what infrastructure exists; stored remotely for team collaboration |

---

### CloudFormation

**AWS CloudFormation** is AWS's native IaC service.

**Terraform vs CloudFormation:**

| Feature | Terraform | CloudFormation |
|---------|-----------|---------------|
| **Cloud support** | Multi-cloud (AWS, GCP, Azure, 1000+ providers) | AWS only |
| **Language** | HCL (HashiCorp Configuration Language) | YAML or JSON |
| **State management** | External (S3 + DynamoDB) | Managed by AWS (automatic) |
| **Drift detection** | `terraform plan` (manual) | Built-in drift detection |
| **Rollback** | Manual (apply previous state) | Automatic rollback on failure |
| **Best for** | Multi-cloud, hybrid | AWS-only environments |

---

### Ansible

**Ansible** (Red Hat) is a configuration management and automation tool. Unlike Terraform (which creates infrastructure), Ansible **configures** existing infrastructure.

> **Ruby Context:** Ansible can be used to configure servers for Rails deployment — installing Ruby, Nginx, and system dependencies. However, with Docker/Kamal, Ansible is less necessary since the container includes all dependencies.

```yaml
# playbook.yml — Configure a server for Rails (non-Docker deployment)
- hosts: web_servers
  become: yes
  tasks:
    - name: Install Ruby dependencies
      apt:
        name: [build-essential, libpq-dev, libssl-dev, zlib1g-dev, git, curl]
        state: present
        update_cache: yes

    - name: Install Ruby via rbenv
      shell: |
        curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash
        rbenv install 3.3.0
        rbenv global 3.3.0
      args:
        creates: /home/deploy/.rbenv/versions/3.3.0

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Copy Nginx config for Rails
      template:
        src: nginx-rails.conf.j2
        dest: /etc/nginx/sites-available/myapp
      notify: Restart Nginx

    - name: Enable site
      file:
        src: /etc/nginx/sites-available/myapp
        dest: /etc/nginx/sites-enabled/myapp
        state: link

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

**Common Pattern:** Use **Terraform** to create infrastructure (VPCs, RDS, ElastiCache, EC2/EKS) and **Kamal** or **Kubernetes** to deploy the Rails application.

---

### Immutable Infrastructure

**Mutable Infrastructure (traditional):**
Servers are updated in place — Capistrano deploys new code, runs migrations, restarts Puma. Over time, servers "drift."

**Immutable Infrastructure (modern):**
Servers are **never modified** after creation. Build a new Docker image, deploy new containers, terminate old ones.

```
Mutable (Capistrano):
  Server created → install Ruby → deploy v1 → deploy v2 → patch OS → deploy v3
  (Server accumulates changes over months → "snowflake")

Immutable (Docker + Kamal/K8s):
  Build image with v1 → deploy containers → build NEW image with v2 → deploy NEW containers → terminate old
  (Every container is identical, created from the same image, never modified)
```

| Feature | Mutable (Capistrano) | Immutable (Docker) |
|---------|---------|-----------|
| **Updates** | Deploy code to existing servers | Replace with new containers |
| **Consistency** | Drift over time | Always consistent (from image) |
| **Rollback** | Complex (revert code, maybe gems changed) | Simple (deploy previous image) |
| **Debugging** | Hard (each server may be different) | Easy (all containers are identical) |
| **Tools** | Capistrano, Ansible | Docker, Kamal, Kubernetes |

**Key Point:** Docker and Kubernetes naturally enforce immutable infrastructure — you don't SSH into a container to update it; you build a new image and deploy it.

---


## 31.5 Cloud Services (Overview)

> Modern system design relies heavily on cloud services. Instead of managing your own servers, databases, and networking, you use managed services from AWS, GCP, or Azure. Understanding the key services and when to use each is essential for system design interviews.

---

### AWS Services Map

```
AWS Services for a Rails Application:

  ┌─────────────────────────────────────────────────────────────────┐
  │                        COMPUTE                                  │
  │  EC2 (Puma/Sidekiq)  │  Lambda  │  ECS/EKS (Containers)       │
  │  Fargate (Serverless containers)                                │
  ├─────────────────────────────────────────────────────────────────┤
  │                        STORAGE                                  │
  │  S3 (ActiveStorage)  │  EBS (DB volumes)  │  EFS (Shared)     │
  ├─────────────────────────────────────────────────────────────────┤
  │                        DATABASE                                 │
  │  RDS/Aurora (PostgreSQL)  │  DynamoDB  │  ElastiCache (Redis)  │
  ├─────────────────────────────────────────────────────────────────┤
  │                        NETWORKING                               │
  │  VPC  │  Route 53 (DNS)  │  CloudFront (CDN)  │  ALB          │
  ├─────────────────────────────────────────────────────────────────┤
  │                        MESSAGING                                │
  │  SQS (Shoryuken)  │  SNS  │  EventBridge  │  MSK (Kafka)     │
  ├─────────────────────────────────────────────────────────────────┤
  │                        SECURITY                                 │
  │  IAM  │  KMS  │  Secrets Manager  │  WAF  │  ACM (TLS)       │
  ├─────────────────────────────────────────────────────────────────┤
  │                        OBSERVABILITY                            │
  │  CloudWatch  │  X-Ray  │  CloudTrail                           │
  └─────────────────────────────────────────────────────────────────┘
```

---

### Compute

| Service | Type | Use Case | Scaling |
|---------|------|----------|---------|
| **EC2** | Virtual machines | Puma/Sidekiq on bare instances; full OS control | Auto Scaling Groups |
| **Lambda** | Serverless functions | Event-driven tasks; S3 triggers; scheduled jobs | Automatic (per-request) |
| **ECS** | Container orchestration (AWS-native) | Docker containers without Kubernetes complexity | Service auto-scaling |
| **EKS** | Managed Kubernetes | Kubernetes workloads; multi-cloud portability | HPA, Cluster Autoscaler |
| **Fargate** | Serverless containers | Containers without managing servers | Automatic |

> **Ruby Context:** Common Rails deployment targets on AWS:

| Scenario | Service | Notes |
|----------|---------|-------|
| Simple Rails app | EC2 + Kamal | Easiest; Kamal deploys Docker containers via SSH |
| PaaS experience | Fargate + ECS | No server management; auto-scaling |
| Kubernetes | EKS + Fargate | Full K8s; most flexible; most complex |
| Serverless API | Lambda + API Gateway | Via `lamby` gem; cold starts are a concern for Ruby |
| Background jobs | EC2/ECS (Sidekiq) | Sidekiq needs persistent Redis connection |

**Lambda vs Containers for Ruby:**

| Feature | Lambda (Ruby) | Containers (ECS/EKS) |
|---------|--------|---------------------|
| Max execution time | 15 minutes | Unlimited |
| Cold start | 1-5 seconds (Ruby is slower than Go/Python) | None (always running) |
| Scaling | Instant (per-request) | Seconds-minutes |
| Cost model | Pay per invocation | Pay for running containers |
| Rails support | Limited (via `lamby` gem) | Full Rails support |
| Best for | Simple functions, webhooks, cron | Full Rails apps, Sidekiq workers |

---

### Storage

| Service | Type | Use Case (Rails) |
|---------|------|----------|
| **S3** | Object storage | ActiveStorage uploads, backups, static assets, data exports |
| **EBS** | Block storage | RDS database volumes, Redis persistence |
| **EFS** | File storage (NFS) | Shared storage across ECS tasks (rare for Rails) |
| **S3 Glacier** | Archive storage | Old backups, compliance archives |

---

### Database

| Service | Type | Use Case (Rails) |
|---------|------|----------|
| **RDS PostgreSQL** | Managed relational DB | Primary database for most Rails apps |
| **Aurora PostgreSQL** | AWS-optimized relational | High-performance; 3x PostgreSQL throughput; auto-scaling replicas |
| **DynamoDB** | Managed NoSQL | Session storage, feature flags, high-scale key-value |
| **ElastiCache Redis** | Managed cache | Rails.cache, Sidekiq broker, Action Cable, session store |
| **ElastiCache Memcached** | Managed cache | Simple caching (Dalli gem) |
| **OpenSearch** | Managed Elasticsearch | Full-text search (Searchkick/Elasticsearch-Rails gems) |
| **Redshift** | Data warehouse | Analytics, BI (query via `ruby-odbc` or export to S3) |

---

### Networking

| Service | Purpose | Rails Use Case |
|---------|---------|----------|
| **VPC** | Isolated virtual network | Network isolation for all Rails infrastructure |
| **Route 53** | DNS + health checks | Domain management, DNS failover |
| **CloudFront** | CDN | Static assets from S3, API caching |
| **ALB** | L7 load balancer | Route HTTP/HTTPS to Puma pods/instances |
| **NLB** | L4 load balancer | TCP for non-HTTP services |
| **API Gateway** | Managed API gateway | Lambda-based Ruby APIs |
| **ACM** | TLS certificates | Free TLS certs for ALB/CloudFront |

---

### Messaging

| Service | Type | Ruby Gem | Use Case |
|---------|------|----------|----------|
| **SQS** | Message queue | `shoryuken`, `aws-sdk-sqs` | Background jobs (alternative to Sidekiq+Redis) |
| **SNS** | Pub/Sub | `aws-sdk-sns` | Notifications, event fan-out |
| **EventBridge** | Event bus | `aws-sdk-eventbridge` | Event-driven architectures |
| **MSK** | Managed Kafka | `karafka`, `rdkafka-ruby` | Event streaming |
| **MQ** | Managed RabbitMQ | `bunny`, `sneakers` | Traditional messaging |

**SQS vs SNS vs EventBridge:**

| Feature | SQS | SNS | EventBridge |
|---------|-----|-----|-------------|
| Pattern | Queue (point-to-point) | Pub/Sub (fan-out) | Event bus (routing) |
| Consumers | One consumer per message | Multiple subscribers | Multiple targets with filtering |
| Ordering | FIFO (optional) | No ordering guarantee | No ordering guarantee |
| Filtering | No | Attribute-based | Content-based (powerful) |
| Best for | Task queues, decoupling | Notifications, fan-out | Event-driven architectures |

---

### Rails Deployment Platform Comparison

| Platform | Complexity | Cost | Scaling | Best For |
|----------|-----------|------|---------|----------|
| **Heroku** | Lowest | High (per dyno) | Auto (paid) | Prototypes, small apps, startups |
| **Render** | Low | Moderate | Auto | Modern Heroku alternative |
| **Fly.io** | Low-Medium | Moderate | Auto | Edge deployment, global apps |
| **Kamal + EC2** | Medium | Low (EC2 pricing) | Manual | Small-medium production apps |
| **ECS + Fargate** | Medium-High | Moderate | Auto (service scaling) | Medium-large apps, AWS-native |
| **EKS** | High | Moderate-High | Auto (HPA) | Large-scale, multi-service |
| **Capistrano + EC2** | Medium | Low | Manual | Legacy Rails apps |

---

## Summary & Key Takeaways

| Topic | Key Point |
|-------|-----------|
| Containers | Package app + dependencies; consistent across environments; Docker is the standard |
| Container vs VM | Containers: lightweight (MBs), fast startup (seconds); VMs: stronger isolation (GBs, minutes) |
| Dockerfile | Multi-stage builds for small images; don't run as root; cache Gemfile layer; use ruby:slim |
| Docker Compose | Multi-container local development; Rails + Sidekiq + PostgreSQL + Redis |
| Kubernetes | Container orchestration; separate Deployments for Puma and Sidekiq; Jobs for migrations |
| K8s Deployment | Manages ReplicaSet; rolling updates; rollback with `kubectl rollout undo` |
| K8s Service | Stable endpoint for pods; ClusterIP (internal), LoadBalancer (external) |
| ConfigMaps/Secrets | Configuration and sensitive data; use External Secrets Operator for production |
| Ingress | HTTP routing rules; path-based routing; TLS termination |
| CI | GitHub Actions with RuboCop, RSpec, Brakeman, bundler-audit; cache gems; parallel tests |
| CD | Continuous Delivery (manual deploy gate) vs Continuous Deployment (auto deploy) |
| Rolling Deployment | Gradual replacement; zero downtime; Kubernetes default |
| Blue-Green | Two environments; instant switch; instant rollback; 2x infrastructure cost |
| Canary | Small % of traffic to new version; data-driven rollout; lowest risk |
| Kamal | Rails-native deployment; Docker containers via SSH; Traefik load balancer; bundled with Rails 8+ |
| Feature Flags | Flipper gem; decouple deployment from release; gradual rollout; instant kill switch |
| Terraform | Multi-cloud IaC; provision RDS, ElastiCache, S3, VPC for Rails |
| Immutable Infrastructure | Docker containers enforce this; never modify running containers; build new image |
| AWS Compute | EC2 + Kamal (simple), ECS/Fargate (managed), EKS (Kubernetes), Lambda (serverless) |
| AWS Database | RDS/Aurora PostgreSQL, ElastiCache Redis, DynamoDB, OpenSearch |
| AWS Messaging | SQS (Shoryuken), SNS, EventBridge, MSK (Karafka) |

**Ruby-Specific Takeaways:**

| Area | Ruby Tools & Gems |
|------|-------------------|
| **Dockerfile** | Rails 7.1+ `bin/rails generate dockerfile`, multi-stage with `ruby:slim` |
| **Local Development** | Docker Compose (Rails + Sidekiq + PostgreSQL + Redis) |
| **Deployment (simple)** | Kamal (Rails 8+ default), Capistrano (legacy) |
| **Deployment (containers)** | ECS/Fargate, EKS, Fly.io |
| **Deployment (PaaS)** | Heroku, Render, Fly.io |
| **CI/CD** | GitHub Actions (`ruby/setup-ruby`, `bundler-cache`), GitLab CI |
| **Linting** | RuboCop, Standard Ruby |
| **Security Scanning** | Brakeman (static analysis), bundler-audit (dependency CVEs) |
| **Testing** | RSpec, Minitest, parallel_tests |
| **Feature Flags** | Flipper (Rails-native), LaunchDarkly |
| **IaC** | Terraform (AWS/GCP/Azure), CloudFormation (AWS-only) |
| **Configuration** | Ansible (server config), Kamal (container deployment) |
| **Migrations (K8s)** | Kubernetes Job (PreSync hook), init containers |

---

## Interview Tips for Module 31

1. **Containers** — explain the difference between containers and VMs; show a Rails multi-stage Dockerfile; mention `ruby:slim` base image
2. **Kubernetes** — draw the architecture; explain Pod, Deployment, Service, Ingress; show separate Deployments for Puma and Sidekiq
3. **K8s Migrations** — explain running `rails db:migrate` as a Kubernetes Job before deployment (PreSync hook)
4. **CI/CD pipeline** — draw the stages; show a GitHub Actions workflow for Rails (RuboCop → RSpec → Brakeman → Docker build)
5. **Deployment strategies** — explain rolling, blue-green, canary with trade-offs; canary is the safest for production
6. **Kamal** — explain as Rails-native deployment; Docker containers via SSH; Traefik for load balancing; compare with Kubernetes
7. **Feature flags** — explain Flipper gem; decouple deployment from release; gradual rollout and kill switch
8. **Terraform** — explain declarative IaC; show provisioning RDS, ElastiCache, S3 for a Rails app
9. **Immutable infrastructure** — explain why Docker is better than Capistrano for consistency; containers enforce immutability
10. **AWS services** — know the key services per category; map them to Rails needs (RDS for DB, ElastiCache for Redis, S3 for uploads)
11. **Lambda vs Containers** — Lambda has cold start issues for Ruby; containers are better for full Rails apps
12. **Platform comparison** — Heroku (simplest), Kamal (Rails-native), ECS (managed), EKS (full K8s); pick based on team size and complexity