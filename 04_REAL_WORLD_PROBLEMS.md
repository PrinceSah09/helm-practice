# Real-World Helm Problems and Solutions

Five problems you will actually face. Each one has full working code, not just explanations.

---

## Problem 1 — Deploy the Same App to Three Environments

### Context

You have a Node.js API. It needs to run in dev, staging, and production. The differences between environments are:

```
dev       1 replica, debug logging, no TLS, dev database
staging   2 replicas, info logging, TLS, staging database
prod      5 replicas, error logging, TLS, production database
```

Without Helm, you maintain three copies of YAML. Every change to the deployment structure needs to be made in three places.

### Solution — values file per environment

The principle: one set of templates, one base `values.yaml`, and small override files for each environment containing only the differences.

#### Chart structure

```
my-api/
├── Chart.yaml
├── values.yaml           <- shared defaults, true for all environments
├── values-dev.yaml       <- only the keys that differ in dev
├── values-staging.yaml   <- only the keys that differ in staging
├── values-prod.yaml      <- only the keys that differ in prod
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

#### values.yaml — shared defaults

```yaml
# values.yaml
# This is the base. Every environment inherits everything here.
# Keep this as the "sensible safe default" — low replicas, no TLS, etc.

replicaCount: 1

image:
  repository: my-registry/my-api
  tag: "latest"   # overridden by CI/CD with the actual git SHA

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

#### values-dev.yaml — dev overrides only

```yaml
# values-dev.yaml
# Only what is different from values.yaml.
# Helm merges this on top of values.yaml — you do not repeat shared keys.

config:
  logLevel: "debug"    # more verbose in dev
  dbHost: "postgres-dev-svc"

ingress:
  enabled: true
  host: "dev-api.internal"
  # tls stays false (inherited from base)
```

#### values-staging.yaml

```yaml
# values-staging.yaml

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
# values-prod.yaml

replicaCount: 5

config:
  logLevel: "error"   # only log errors in prod to reduce noise
  dbHost: "postgres-prod-svc"

ingress:
  enabled: true
  tls: true
  host: "api.example.com"

# prod gets more resources than the base
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 2Gi
```

#### templates/deployment.yaml — reads from values

```yaml
# templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-api.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: api
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          ports:
            - containerPort: {{ .Values.service.port }}
          env:
            # LOG_LEVEL comes from config.logLevel in values
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            # DB_HOST points to the correct database for this environment
            - name: DB_HOST
              value: {{ .Values.config.dbHost | quote }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

#### Deploy commands

```bash
# Deploy to dev
helm upgrade --install my-api ./my-api \
  -f my-api/values.yaml \
  -f my-api/values-dev.yaml \
  -n dev --create-namespace

# Deploy to staging
helm upgrade --install my-api ./my-api \
  -f my-api/values.yaml \
  -f my-api/values-staging.yaml \
  -n staging --create-namespace

# Deploy to prod
helm upgrade --install my-api ./my-api \
  -f my-api/values.yaml \
  -f my-api/values-prod.yaml \
  -n prod --create-namespace
```

#### How the merge works

```
values.yaml (base)
  replicaCount: 1
  config.logLevel: "info"
  config.dbHost:   "localhost"
  ingress.enabled: false
       +
values-prod.yaml (overrides)
  replicaCount: 5             <- replaces 1
  config.logLevel: "error"    <- replaces "info"
  config.dbHost:   "prod-svc" <- replaces "localhost"
  ingress.enabled: true       <- replaces false
       =
Result in prod:
  replicaCount: 5
  config.logLevel: "error"
  config.dbHost:   "prod-svc"
  ingress.enabled: true
  resources: (unchanged, from base)
```

> One chart. Three environments. Change the Deployment template once and all three environments get the update on next deploy.

---

## Problem 2 — Deploy a New Version Without Any Downtime

### Context

Your API is running in production with 5 replicas at v1.0.0. You need to ship v2.0.0. Requirements:
- No user-visible errors during the deploy
- Database migrations must run before new pods start
- If anything goes wrong, automatic rollback in under 60 seconds

### Solution — pre-upgrade hook + rolling update + --atomic

#### Step 1: Database migration as a pre-upgrade hook

```yaml
# templates/pre-upgrade-migration.yaml
# This Job runs BEFORE the Deployment is updated.
# If the Job fails, the upgrade is aborted — new pods never start.

apiVersion: batch/v1
kind: Job
metadata:
  # Include the revision number so each upgrade creates a uniquely named Job.
  # Without this, the second upgrade would fail because the Job already exists.
  name: {{ include "my-api.fullname" . }}-migration-{{ .Release.Revision }}
  annotations:
    "helm.sh/hook": pre-upgrade

    # Weight controls ordering when you have multiple hooks.
    # Lower number runs first. -5 means "run early."
    "helm.sh/hook-weight": "-5"

    # Clean up the Job before creating a new one on the next upgrade,
    # and also clean up after it succeeds to keep the cluster tidy.
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3    # retry up to 3 times if the migration script exits non-zero
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migration
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
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

#### Step 2: Deployment with rolling update and readiness probe

```yaml
# templates/deployment.yaml (relevant sections)

spec:
  replicas: {{ .Values.replicaCount }}

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0   # never allow fewer than desired replicas during update
      maxSurge: 1         # allow one extra pod above desired count during update

  template:
    spec:
      containers:
        - name: api
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}

          # CRITICAL: Kubernetes will not route traffic to a pod until
          # the readiness probe passes. This is what makes zero-downtime work.
          # Without this, a pod can start receiving traffic before it is ready.
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5    # wait 5s before first check
            periodSeconds: 5          # check every 5s

          # Liveness probe restarts the pod if it becomes unresponsive
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
```

#### Step 3: The upgrade command

```bash
# --atomic: if pods do not become healthy within the timeout, rollback automatically
# --wait:   block until all pods pass their readiness probes
# Together these give you zero-downtime deploys with automatic safety net

helm upgrade my-api ./my-api \
  -f values-prod.yaml \
  --set image.tag=v2.0.0 \
  --atomic \
  --wait \
  --timeout 10m
```

#### What happens end to end

```
helm upgrade --atomic ... --set image.tag=v2.0.0
      |
      v
  1. Pre-upgrade hook: migration Job starts
     Helm waits for the Job to complete successfully.
     If it fails: upgrade is aborted, cluster unchanged.
      |
      v
  2. Rolling update begins
     [v1][v1][v1][v1][v1]   initial state

     Start one new pod:
     [v1][v1][v1][v1][v1][v2]

     Wait for v2 to pass readiness probe.
     Then terminate one v1:
     [v1][v1][v1][v1][v2]

     Continue until:
     [v2][v2][v2][v2][v2]   done, zero downtime

  3. All pods healthy -> upgrade marked "deployed"

  If any pod fails to become ready within 10 minutes:
      |
      v
  4. --atomic triggers automatic rollback to v1.0.0
     Revision 7 is added to history: "Rollback to 6"
```

> The combination of readiness probes + rolling update guarantees traffic only reaches ready pods. The --atomic flag guarantees you never leave the cluster in a broken state.

---

## Problem 3 — Eight Microservices, One Chart

### Context

You have eight microservices, all structurally the same (Deployment + Service + Ingress + HPA). Writing and maintaining eight separate charts is duplicate work. Any structural change (new probe, updated resource limits policy) requires editing eight charts.

```
user-service      Node.js   port 3001
order-service     Node.js   port 3002
payment-service   Node.js   port 3003
notification-svc  Python    port 3004
inventory-service Go        port 3005
search-service    Python    port 3006
auth-service      Node.js   port 3007
gateway-service   Go        port 3008
```

### Solution — one generic chart, per-service values files

Create a single `microservice` chart. Deploy it eight times with different values.

#### The generic chart — values.yaml

```yaml
# microservice/values.yaml
# Every configurable aspect of a microservice is a value here.
# Services that do not need a feature simply leave it disabled.

# required: set this to the service name
name: ""

replicaCount: 2

image:
  repository: ""    # required: your container image
  tag: "latest"

service:
  port: 8080
  targetPort: 8080

# Ingress is off by default, enabled per service as needed
ingress:
  enabled: false
  host: ""
  path: "/"
  tls: false

# Arbitrary env vars — each service declares its own
# Example:
# env:
#   - name: DB_HOST
#     value: postgres-svc
env: []

# Reference existing K8s Secrets as env vars
# Example:
# envFrom:
#   - secretRef:
#       name: payment-service-secrets
envFrom: []

# HPA is off by default, enabled for high-traffic services
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Health check endpoint — all services must implement this
healthCheck:
  path: /health
  port: 8080
```

#### Per-service values files

```yaml
# values-user-service.yaml
name: user-service
image:
  repository: my-registry/user-service
  tag: "1.4.2"
service:
  port: 3001
  targetPort: 3001
env:
  - name: DB_HOST
    value: "user-db-svc"
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
healthCheck:
  path: /health
  port: 3001
```

```yaml
# values-payment-service.yaml
name: payment-service
image:
  repository: my-registry/payment-service
  tag: "3.0.1"
service:
  port: 3003
  targetPort: 3003
env:
  - name: DB_HOST
    value: "payment-db-svc"
  - name: STRIPE_ENDPOINT
    value: "https://api.stripe.com"
# Payment service gets more CPU — it does heavy crypto work
resources:
  requests:
    cpu: 500m
    memory: 256Mi
  limits:
    cpu: 2000m
    memory: 512Mi
envFrom:
  - secretRef:
      name: payment-service-secrets   # contains STRIPE_SECRET_KEY
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
healthCheck:
  path: /health
  port: 3003
```

#### Deploy all eight from the same chart

```bash
# One chart, eight releases, each with its own values file
helm upgrade --install user-service      ./microservice -f values-user-service.yaml      -n prod
helm upgrade --install order-service     ./microservice -f values-order-service.yaml     -n prod
helm upgrade --install payment-service   ./microservice -f values-payment-service.yaml   -n prod
helm upgrade --install notification-svc  ./microservice -f values-notification-svc.yaml  -n prod
helm upgrade --install inventory-service ./microservice -f values-inventory-service.yaml -n prod
helm upgrade --install search-service    ./microservice -f values-search-service.yaml    -n prod
helm upgrade --install auth-service      ./microservice -f values-auth-service.yaml      -n prod
helm upgrade --install gateway-service   ./microservice -f values-gateway-service.yaml   -n prod
```

#### What you get

```
microservice/ chart (one copy)
      |
      |-- -f values-user-service.yaml     -> user-service Deployment + Service + HPA
      |-- -f values-order-service.yaml    -> order-service Deployment + Service
      |-- -f values-payment-service.yaml  -> payment-service Deployment + Service + HPA
      |-- -f values-auth-service.yaml     -> auth-service Deployment + Service + Ingress
      ...

Add a new liveness probe policy?
  Edit microservice/templates/deployment.yaml once.
  Redeploy all eight services.
  Done.
```

> This is the most common Helm pattern in teams with multiple microservices. The chart is the shared infrastructure template. The values files are per-team configuration.

---

## Problem 4 — Production Outage, Need Immediate Rollback

### Context

```
16:00  helm upgrade deployed v3.1.0 to production
16:03  error rate jumps from 0.1% to 40% in Grafana
16:05  users are reporting failures, Slack is loud
16:06  you need to rollback NOW
```

### Solution — one command

```bash
# Step 1: check what was running before this version
helm history my-api -n prod

# Output:
# REVISION  UPDATED       STATUS      APP VER   DESCRIPTION
# 10        16:00 Fri     superseded  v3.0.0    Upgrade complete   <- this worked
# 11        16:03 Fri     deployed    v3.1.0    Upgrade complete   <- this is broken

# Step 2: rollback immediately
# --wait blocks until pods are healthy so you know when it is done
helm rollback my-api 10 -n prod --wait

# Takes 30-60 seconds. Kubernetes does a rolling rollback.
```

### What helm rollback actually restores

This is the important part — it is not just the image tag:

```
helm rollback my-api 10 restores the exact state of EVERY resource at revision 10:

  Deployment     image tag, env vars, resource limits, probe config
  Service        ports, type, annotations
  ConfigMap      all key-value pairs
  Ingress        hosts, paths, TLS config
  HPA            min/max replicas, CPU targets

Nothing is missed. kubectl rollout undo only handles the Deployment —
it would leave a broken ConfigMap or Service in place.
```

### Confirm the rollback worked

```bash
# Check the release status
helm status my-api -n prod
# Should show STATUS: deployed

# A new revision was created — rollback is non-destructive
helm history my-api -n prod
# REVISION  STATUS      DESCRIPTION
# 10        superseded  Upgrade complete
# 11        superseded  Upgrade failed   (or deployed, depending on timing)
# 12        deployed    Rollback to 10

# Verify pods are healthy
kubectl get pods -n prod
kubectl rollout status deployment/my-api -n prod

# Compare the broken version to the working one
helm get manifest my-api --revision 11 > /tmp/broken.yaml
helm get manifest my-api --revision 10 > /tmp/working.yaml
diff /tmp/working.yaml /tmp/broken.yaml
```

### How to prevent this from being manual

```bash
# Add --atomic to every production upgrade.
# This makes rollback automatic — you never have to intervene.

helm upgrade my-api ./my-api \
  --set image.tag=v3.1.0 \
  -f values-prod.yaml \
  --atomic \          # if pods fail health checks, auto-rollback
  --wait \
  --timeout 5m

# If v3.1.0 pods crash or fail readiness within 5 minutes:
# Error: UPGRADE FAILED: release my-api failed, and has been rolled back
# Revision 12 is automatically created: "Rollback to 10"
```

### Keep enough history to fall back to

```bash
# Default history limit is 10. In production, keep more.
helm upgrade my-api ./my-api --history-max 30

# Never truncate history on the upgrade that just broke things.
# If history is full and gets trimmed, you lose the revision you need.
```

> The pattern: always use --atomic in CI/CD, always keep at least 20 revisions of history, and know this command by heart: `helm rollback <release> <revision> -n <namespace> --wait`

---

## Problem 5 — App Needs PostgreSQL and Redis Without Writing Your Own Charts

### Context

You are building a Django application. It needs:
- Your custom Django app
- PostgreSQL for the database
- Redis for caching and Celery task queue

You could write K8s StatefulSets and PersistentVolumeClaims from scratch, or you can use Helm chart dependencies and get battle-tested setups from Bitnami in minutes.

### Solution — chart dependencies

#### Chart.yaml — declare what you need

```yaml
# Chart.yaml

apiVersion: v2
name: my-django-app
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "12.1.2"
    repository: "https://charts.bitnami.com/bitnami"
    # Only install postgres if this value is true.
    # In prod, we use AWS RDS, so we set this to false in values-prod.yaml.
    condition: postgresql.enabled

  - name: redis
    version: "17.3.11"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

#### values.yaml — configure everything in one file

```yaml
# values.yaml

# Your Django app
replicaCount: 2
image:
  repository: my-registry/my-django-app
  tag: "1.0.0"

env:
  - name: DATABASE_URL
    # The hostname follows the pattern: <release-name>-postgresql
    value: "postgresql://myuser:changeme@my-django-app-postgresql:5432/mydb"
  - name: REDIS_URL
    # The hostname follows the pattern: <release-name>-redis-master
    value: "redis://my-django-app-redis-master:6379/0"

# -- PostgreSQL sub-chart configuration ------------------------------------
# Keys under "postgresql:" are passed directly to the bitnami/postgresql chart.
# Run "helm show values bitnami/postgresql" to see all available options.
postgresql:
  enabled: true
  auth:
    username: myuser
    password: "changeme"   # never hardcode this in real deployments — use helm-secrets
    database: mydb
  primary:
    persistence:
      enabled: true
      size: 20Gi

# -- Redis sub-chart configuration -----------------------------------------
# Keys under "redis:" are passed to the bitnami/redis chart.
redis:
  enabled: true
  auth:
    enabled: false       # no password — Redis is only accessible inside the cluster
  master:
    persistence:
      enabled: false     # Redis is a cache, not a database — no persistence needed
  replica:
    replicaCount: 0      # zero replicas in dev to save resources
```

#### Download dependencies, then deploy

```bash
# Download the sub-charts into the charts/ directory
# This also creates Chart.lock (like package-lock.json)
helm dependency update ./my-django-app

# Deploy everything with one command:
# your app + postgres + redis
helm install my-app ./my-django-app -f values.yaml -n dev --create-namespace
```

#### What gets created

```
helm install my-app ./my-django-app
      |
      |-- my-app-deployment              your Django app (Deployment + Service)
      |
      |-- my-app-postgresql              full Postgres from bitnami
      |       |-- StatefulSet
      |       |-- Service (headless + regular)
      |       |-- PersistentVolumeClaim
      |       +-- Secret (DB credentials)
      |
      +-- my-app-redis                   full Redis from bitnami
              |-- Deployment
              |-- Service
              +-- ConfigMap
```

#### Pre-upgrade hook: run Django migrations before new pods start

```yaml
# templates/pre-upgrade-migration.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-django-app.fullname" . }}-migrate-{{ .Release.Revision }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      initContainers:
        # Wait for Postgres to be accepting connections before running migrate.
        # Without this, the migration Job would fail on first install because
        # postgres is still starting up.
        - name: wait-for-postgres
          image: busybox
          command:
            - sh
            - -c
            - |
              until nc -z {{ .Release.Name }}-postgresql 5432; do
                echo "waiting for postgres..."
                sleep 2
              done
              echo "postgres is ready"
      containers:
        - name: migrate
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["python", "manage.py", "migrate", "--noinput"]
          env:
            - name: DATABASE_URL
              value: "postgresql://myuser:changeme@{{ .Release.Name }}-postgresql:5432/mydb"
```

#### Prod: disable the in-cluster Postgres, use RDS instead

```yaml
# values-prod.yaml
# In production, the database is AWS RDS — more reliable, managed backups, etc.
# We disable the postgres sub-chart and point the app at the external host.

postgresql:
  enabled: false   # do not install bitnami/postgresql in the cluster

redis:
  enabled: true
  replica:
    replicaCount: 2   # in prod, run replicas for availability

# Override the DATABASE_URL to point at AWS RDS
env:
  - name: DATABASE_URL
    value: "postgresql://myuser@prod-db.us-east-1.rds.amazonaws.com:5432/mydb"
  - name: REDIS_URL
    value: "redis://my-django-app-redis-master:6379/0"
```

```bash
# dev: app + in-cluster postgres + redis
helm upgrade --install my-app ./my-django-app -f values-dev.yaml -n dev

# prod: app + redis only (postgres is RDS)
helm upgrade --install my-app ./my-django-app -f values-prod.yaml -n prod
```

> Do not write your own Postgres StatefulSet. Bitnami's chart is maintained by a large team, tested across Kubernetes versions, and handles edge cases you would spend weeks finding. Use dependencies for infrastructure components, write charts only for your own applications.

---

## Bonus — The Debugging Checklist

When a release is broken, follow this sequence:

```bash
# 1. What is the release status?
helm status my-app -n prod

# 2. What values are currently active?
helm get values my-app -n prod --all

# 3. What YAML was applied?
helm get manifest my-app -n prod

# 4. Are pods running?
kubectl get pods -n prod

# 5. Why is a specific pod failing?
kubectl describe pod <pod-name> -n prod   # look at the Events section
kubectl logs <pod-name> -n prod
kubectl logs <pod-name> -n prod --previous   # logs from crashed previous instance

# 6. What changed between the working and broken revision?
helm get manifest my-app --revision 10 > /tmp/old.yaml
helm get manifest my-app --revision 11 > /tmp/new.yaml
diff /tmp/old.yaml /tmp/new.yaml

# 7. Rollback if you cannot fix it quickly
helm rollback my-app -n prod --wait
```

---

> Back to: [02_HELM_COMPLETE_GUIDE.md](./02_HELM_COMPLETE_GUIDE.md) | [03_HELM_CHEATSHEET.md](./03_HELM_CHEATSHEET.md)
