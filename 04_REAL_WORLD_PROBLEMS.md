# 🔥 Real-World Helm Problems & Solutions

> These are the exact problems you'll hit in production. Each one has context, the problem, the solution, and the lesson.

---

## Problem 1: Deploy the Same App to Dev, Staging & Production

### The Situation

You have a Node.js API. You need it running in 3 environments:

```
dev       → 1 replica, no TLS, debug logging, connects to dev DB
staging   → 2 replicas, TLS, info logging, connects to staging DB
prod      → 5 replicas, TLS, error logging, connects to prod DB
```

Without Helm, you maintain 3 copies of YAML. That's a maintenance nightmare.

---

### The Solution — Environment Values Files

#### Chart structure

```
my-api/
├── Chart.yaml
├── values.yaml            ← shared defaults
├── values-dev.yaml        ← dev overrides
├── values-staging.yaml    ← staging overrides
├── values-prod.yaml       ← prod overrides
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

#### values.yaml (shared defaults)

```yaml
replicaCount: 1

image:
  repository: my-registry/my-api
  tag: "latest"

service:
  type: ClusterIP
  port: 3000

ingress:
  enabled: false
  tls: false
  host: ""

config:
  logLevel: "info"
  dbHost: "localhost"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

#### values-dev.yaml

```yaml
# ONLY what's different from defaults
replicaCount: 1

config:
  logLevel: "debug"
  dbHost: "postgres-dev-svc"

ingress:
  enabled: true
  host: "dev-api.internal"
```

#### values-staging.yaml

```yaml
replicaCount: 2

config:
  logLevel: "info"
  dbHost: "postgres-staging-svc"

ingress:
  enabled: true
  tls: true
  host: "staging-api.example.com"
```

#### values-prod.yaml

```yaml
replicaCount: 5

config:
  logLevel: "error"
  dbHost: "postgres-prod-svc"

ingress:
  enabled: true
  tls: true
  host: "api.example.com"

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 2Gi
```

#### templates/deployment.yaml (reads from values)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-api.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  ...
  template:
    spec:
      containers:
        - name: api
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          env:
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            - name: DB_HOST
              value: {{ .Values.config.dbHost | quote }}
```

#### Deploy Commands

```bash
# Dev
helm upgrade --install my-api ./my-api \
  -f my-api/values.yaml \
  -f my-api/values-dev.yaml \
  -n dev --create-namespace

# Staging
helm upgrade --install my-api ./my-api \
  -f my-api/values.yaml \
  -f my-api/values-staging.yaml \
  -n staging --create-namespace

# Production
helm upgrade --install my-api ./my-api \
  -f my-api/values.yaml \
  -f my-api/values-prod.yaml \
  -n prod --create-namespace
```

#### Visual — How Merging Works

```
values.yaml (base)
  replicaCount: 1
  config.logLevel: "info"
  config.dbHost: "localhost"
  ingress.enabled: false
         +
values-prod.yaml (overrides)
  replicaCount: 5             ← overrides base
  config.logLevel: "error"    ← overrides base
  config.dbHost: "prod-db"    ← overrides base
  ingress.enabled: true       ← overrides base
         =
Final values for prod:
  replicaCount: 5        ✅
  config.logLevel: error ✅
  config.dbHost: prod-db ✅
  ingress.enabled: true  ✅
```

> **Lesson:** One chart, one template, N environments. Change image tag? Update values.yaml once. Done.

---

## Problem 2: Deploy a New App Version Without Downtime

### The Situation

Your app is running in production. You have a new version `v2.0.0` to deploy. Requirements:
- Zero downtime (users must not see errors)
- Able to rollback in 60 seconds if something breaks
- Must run DB migrations before new pods start

---

### The Solution — Hooks + Atomic Upgrade

#### Step 1: Pre-upgrade hook for DB migration

```yaml
# templates/pre-upgrade-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-api.fullname" . }}-migration-{{ .Release.Revision }}
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3                    # retry up to 3 times
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migration
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["node", "scripts/migrate.js"]
          env:
            - name: DB_HOST
              value: {{ .Values.config.dbHost | quote }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-api.fullname" . }}-secrets
                  key: DB_PASSWORD
```

#### Step 2: Deployment with RollingUpdate strategy

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never kill a pod before new one is ready
      maxSurge: 1          # allow 1 extra pod during rollout
  template:
    spec:
      containers:
        - name: api
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          readinessProbe:            # CRITICAL — pod only gets traffic when truly ready
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
```

#### Step 3: The upgrade command

```bash
helm upgrade my-api ./my-api \
  -f values-prod.yaml \
  --set image.tag=v2.0.0 \
  --atomic \              # auto-rollback if anything fails
  --wait \                # wait for all pods healthy
  --timeout 10m           # give it 10 minutes
```

#### What happens step by step

```
helm upgrade --atomic my-api ./my-api --set image.tag=v2.0.0
      │
      ▼
  1. pre-upgrade hook fires
     Migration Job runs → DB schema updated ✅
      │
      ▼
  2. Rolling update begins
     Old: [v1.0][v1.0][v1.0][v1.0][v1.0]
     
     Step: [v1.0][v1.0][v1.0][v1.0][v1.0][v2.0]  ← new pod starts
     Wait: readinessProbe passes ✅
     Step: [v1.0][v1.0][v1.0][v1.0][v2.0]        ← old pod removed
     ...continues...
     Done: [v2.0][v2.0][v2.0][v2.0][v2.0] ✅

  3. All pods healthy → upgrade marked succeeded
      │
  If ANYTHING fails:
      ▼
  4. --atomic triggers automatic rollback to v1.0.0
     DB migration is backward-compatible so no issues
```

#### Rollback if needed after the fact

```bash
# See what's deployed
helm history my-api

# REVISION  STATUS     APP VER  DESCRIPTION
# 5         deployed   v1.0.0   Install
# 6         failed     v2.0.0   Upgrade failed

# Rollback
helm rollback my-api 5 --wait
```

> **Lesson:** `--atomic` is your safety net. Always use it in production upgrades. Combine with readiness probes — Kubernetes won't route traffic to an unhealthy pod.

---

## Problem 3: One Chart, Many Microservices

### The Situation

You have 8 microservices:
- `user-service`
- `order-service`
- `payment-service`
- `notification-service`
- `inventory-service`
- `search-service`
- `auth-service`
- `gateway-service`

Each is just a Node.js or Python app. Writing 8 separate charts is massive duplication.

---

### The Solution — A Generic "microservice" Library Chart

Create ONE chart that works for any microservice. Just change values.

#### The generic chart structure

```
microservice/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── configmap.yaml
    └── _helpers.tpl
```

#### values.yaml — every knob you might want

```yaml
# Core
name: ""                   # REQUIRED: service name
replicaCount: 2

image:
  repository: ""           # REQUIRED: docker image
  tag: "latest"

# Service
service:
  port: 8080
  targetPort: 8080

# Ingress
ingress:
  enabled: false
  host: ""
  path: "/"

# Env vars — any service can set these
env: []
# - name: DB_HOST
#   value: postgres-svc

# Secrets from K8s Secrets
envFrom: []
# - secretRef:
#     name: my-secret

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# Resources
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Health checks
healthCheck:
  path: /health
  port: 8080
```

#### Deploy each service with its own values file

```bash
# user-service-values.yaml
name: user-service
image:
  repository: my-registry/user-service
  tag: "1.4.2"
service:
  port: 3001
env:
  - name: DB_HOST
    value: "user-db-svc"
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
```

```bash
# order-service-values.yaml
name: order-service
image:
  repository: my-registry/order-service
  tag: "2.1.0"
service:
  port: 3002
env:
  - name: DB_HOST
    value: "order-db-svc"
  - name: PAYMENT_SERVICE_URL
    value: "http://payment-service:3003"
```

```bash
# Deploy all 8 services from the SAME chart
helm upgrade --install user-service      ./microservice -f user-service-values.yaml -n prod
helm upgrade --install order-service     ./microservice -f order-service-values.yaml -n prod
helm upgrade --install payment-service   ./microservice -f payment-service-values.yaml -n prod
# ... and so on
```

#### Visual

```
One Chart "microservice"
        │
        ├──── -f user-service-values.yaml    ──▶ user-service Deployment + Service
        ├──── -f order-service-values.yaml   ──▶ order-service Deployment + Service
        ├──── -f payment-service-values.yaml ──▶ payment-service Deployment + Service
        └──── -f auth-service-values.yaml    ──▶ auth-service Deployment + Service

Shared logic lives in ONE place ✅
Update HPA policy? Edit chart once, redeploy all ✅
```

> **Lesson:** A generic chart + per-service values files is the most scalable Helm pattern for microservices. 1 chart, N configs.

---

## Problem 4: Production Outage — Bad Deploy Needs Instant Rollback

### The Situation

It's Friday 5pm. You deploy `v3.1.0`. Monitoring shows error rate jumps to 40%. Users are screaming. You need to rollback RIGHT NOW.

```
Timeline:
  17:00 — helm upgrade deployed v3.1.0
  17:03 — errors spike in Grafana
  17:05 — Slack is on fire 🔥
  17:06 — YOU NEED TO ROLLBACK
```

---

### The Solution — One Command Rollback

```bash
# Step 1: Find what revision was working
helm history my-api -n prod

# REVISION  UPDATED     STATUS      APP VER   DESCRIPTION
# 10        Fri 16:00   superseded  v3.0.0    Upgrade complete  ← this was working
# 11        Fri 17:00   deployed    v3.1.0    Upgrade complete  ← BROKEN

# Step 2: Rollback immediately
helm rollback my-api 10 -n prod --wait

# This takes ~30-60 seconds
# Kubernetes does a rolling rollback (same as rolling update, in reverse)
```

#### What Helm rollback actually does

```
It doesn't just revert the Deployment image tag.
It restores the EXACT state of EVERY resource from revision 10:
  ✅ Deployment (image, env vars, resources, etc.)
  ✅ Service (ports, type)
  ✅ ConfigMap (all config values)
  ✅ Ingress rules
  ✅ HPA settings
  Everything. A true full rollback.
```

#### Before it gets to this — set up alerting + auto-rollback

```bash
# Prevent this scenario with --atomic on every upgrade
helm upgrade my-api ./my-api \
  --set image.tag=v3.1.0 \
  -f values-prod.yaml \
  --atomic \          # if pods don't become healthy, auto-rollback
  --wait \
  --timeout 5m

# If pods fail health checks within 5 minutes → automatic rollback to v3.0.0
# You get a clear error message:
# Error: UPGRADE FAILED: release my-api failed, and has been rolled back
```

#### Keep a history to fall back to

```bash
# Configure max history in upgrade commands
helm upgrade my-api ./my-api --history-max 20

# Or set a global default in your values
# Never let history get truncated when you need it most
```

#### Post-incident checklist

```bash
# 1. Confirm rollback succeeded
helm status my-api -n prod
helm history my-api -n prod    # revision 12 should be "Rollback to 10"

# 2. Verify pods are healthy
kubectl get pods -n prod
kubectl rollout status deployment/my-api -n prod

# 3. Check error rate dropped
# (check your monitoring — Grafana, Datadog, etc.)

# 4. Inspect what was wrong in v3.1.0
helm get manifest my-api --revision 11 > broken-v3.1.0.yaml
helm get manifest my-api --revision 10 > working-v3.0.0.yaml
diff working-v3.0.0.yaml broken-v3.1.0.yaml
```

> **Lesson:** `helm rollback` in one command beats manual YAML hunting. But the real lesson is `--atomic` — make rollbacks automatic. Fix it before Friday 5pm ever happens.

---

## Problem 5: App Needs PostgreSQL — Don't Write Your Own Chart

### The Situation

You're building a new service that needs:
- Your custom app (Python/Django)
- PostgreSQL database
- Redis cache

You could write K8s YAMLs for postgres and redis from scratch... or you could just use Helm dependencies.

---

### The Solution — Chart Dependencies

#### Chart.yaml — declare what you need

```yaml
apiVersion: v2
name: my-django-app
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "12.1.2"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled

  - name: redis
    version: "17.3.11"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

#### values.yaml — configure everything in one place

```yaml
# Your app
replicaCount: 2
image:
  repository: my-registry/my-django-app
  tag: "1.0.0"

# Environment variables for your app
env:
  - name: DATABASE_URL
    value: "postgresql://myuser:$(DB_PASSWORD)@my-django-app-postgresql:5432/mydb"
  - name: REDIS_URL
    value: "redis://my-django-app-redis-master:6379"
  - name: DJANGO_SECRET_KEY
    valueFrom:
      secretKeyRef:
        name: my-django-app-secrets
        key: DJANGO_SECRET_KEY

# ── PostgreSQL sub-chart config ──────────────────────────────
# These keys go DIRECTLY to the postgresql chart's values
postgresql:
  enabled: true
  auth:
    username: myuser
    password: "changeme"          # use secrets in real life!
    database: mydb
  primary:
    persistence:
      enabled: true
      size: 20Gi
  resources:
    requests:
      cpu: 250m
      memory: 256Mi

# ── Redis sub-chart config ───────────────────────────────────
redis:
  enabled: true
  auth:
    enabled: false                # no password for internal use
  master:
    persistence:
      enabled: false              # stateless for caching
  replica:
    replicaCount: 0               # no replicas in dev (save resources)
```

#### Install it all with ONE command

```bash
# First, download the sub-charts
helm dependency update ./my-django-app

# Deploy everything: your app + postgres + redis
helm install my-app ./my-django-app -f values.yaml -n dev --create-namespace
```

#### What gets created

```
helm install my-app ./my-django-app
      │
      ├──▶ my-app-deployment           (your Django app)
      ├──▶ my-app-service              (service for your app)
      ├──▶ my-app-postgresql           (full postgres from bitnami)
      │       ├── StatefulSet
      │       ├── Service
      │       ├── PersistentVolumeClaim
      │       └── Secret (DB password)
      └──▶ my-app-redis                (full redis from bitnami)
              ├── Deployment
              ├── Service
              └── ConfigMap
```

#### Pre-upgrade hook for Django migrations

```yaml
# templates/pre-upgrade-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-django-app.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      initContainers:
        - name: wait-for-postgres          # don't run until DB is ready
          image: busybox
          command:
            - sh
            - -c
            - |
              until nc -z {{ .Release.Name }}-postgresql 5432; do
                echo "Waiting for postgres..."
                sleep 2
              done
      containers:
        - name: migrate
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["python", "manage.py", "migrate", "--noinput"]
          env:
            - name: DATABASE_URL
              value: "postgresql://myuser:changeme@{{ .Release.Name }}-postgresql:5432/mydb"
```

#### Disable sub-charts for environments that don't need them

```yaml
# values-prod.yaml
# In prod, postgres is managed externally (RDS/Cloud SQL)
postgresql:
  enabled: false          # don't install postgres sub-chart

redis:
  enabled: true
  replica:
    replicaCount: 2       # 2 replicas in prod

# Point your app to external DB
env:
  - name: DATABASE_URL
    value: "postgresql://myuser:$(DB_PASSWORD)@prod-db.rds.amazonaws.com:5432/mydb"
```

```bash
# Dev: deploys app + postgres + redis (all local)
helm upgrade --install my-app ./my-django-app -f values-dev.yaml -n dev

# Prod: deploys app + redis only (postgres is AWS RDS)
helm upgrade --install my-app ./my-django-app -f values-prod.yaml -n prod
```

> **Lesson:** Helm dependencies let you compose complex systems from battle-tested charts. Don't write your own postgres chart. Let the 10,000+ people who maintain `bitnami/postgresql` handle that. You focus on your app.

---

## Bonus: The 30-Second Debugging Flow

When something goes wrong with a Helm release, follow this:

```bash
# 1. What's the release status?
helm status my-app -n prod

# 2. What's actually deployed?
helm get manifest my-app -n prod

# 3. What values are in use?
helm get values my-app -n prod --all

# 4. Are pods running?
kubectl get pods -n prod

# 5. Why is a pod failing?
kubectl describe pod <pod-name> -n prod    # check Events section
kubectl logs <pod-name> -n prod

# 6. What changed in this revision vs last?
helm get manifest my-app --revision 11 > new.yaml
helm get manifest my-app --revision 10 > old.yaml
diff old.yaml new.yaml

# 7. Rollback if needed
helm rollback my-app -n prod --wait
```

---

> **Back to →** [02_HELM_COMPLETE_GUIDE.md](./02_HELM_COMPLETE_GUIDE.md) | [03_HELM_CHEATSHEET.md](./03_HELM_CHEATSHEET.md)
