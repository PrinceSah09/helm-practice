# Helm — The Complete Guide

> Helm is to Kubernetes what apt or brew is to your OS.
> It packages, versions, and manages your K8s applications.

---

## Table of Contents

1. [Why Helm Exists](#1-why-helm-exists)
2. [How Helm Works](#2-how-helm-works)
3. [Architecture](#3-architecture)
4. [Installation](#4-installation)
5. [Repositories](#5-repositories)
6. [Charts — Deep Dive](#6-charts--deep-dive)
7. [Values and Templating](#7-values-and-templating)
8. [Installing Releases](#8-installing-releases)
9. [Upgrading Releases](#9-upgrading-releases)
10. [Rollbacks](#10-rollbacks)
11. [Hooks](#11-hooks)
12. [Chart Dependencies](#12-chart-dependencies)
13. [Packaging and Sharing](#13-packaging-and-sharing)
14. [Testing Charts](#14-testing-charts)
15. [Handling Secrets](#15-handling-secrets)
16. [Things Worth Knowing](#16-things-worth-knowing)

---

## 1. Why Helm Exists

### The raw YAML problem

A production-grade app typically needs all of these:

```
my-app/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── configmap.yaml
├── secret.yaml
├── hpa.yaml
└── pdb.yaml
```

Multiply that across three environments:

```
Without Helm:

  my-app-dev/
    deployment.yaml    <- 90% identical to prod version
    configmap.yaml     <- different DB_HOST
    ...

  my-app-staging/
    deployment.yaml    <- 90% identical to prod version
    ...

  my-app-prod/
    deployment.yaml
    ...

Problems this creates:
  - Same YAML duplicated in three places
  - Change the image tag -> edit three files, risk missing one
  - "What config is currently running in prod?" is a manual audit
  - Rollback means manually reverting YAML files and re-applying
```

### With Helm

```
With Helm:

  my-app/
    templates/              <- one set of templates
      deployment.yaml
      service.yaml
      ...
    values.yaml             <- shared defaults
    values-dev.yaml         <- only the differences for dev
    values-staging.yaml     <- only the differences for staging
    values-prod.yaml        <- only the differences for prod

  # Deploy to prod with one command
  helm upgrade --install my-app ./my-app -f values-prod.yaml

  # Rollback to the previous version
  helm rollback my-app 2
```

One template set. N environments. Full version history. One-command rollback.

---

## 2. How Helm Works

```
+------------------------------------------------------------------+
|                         HELM FLOW                                |
|                                                                  |
|  1. You write templates/ and values.yaml                         |
|                     |                                            |
|                     v                                            |
|  2. helm install my-release ./my-chart -f values-prod.yaml       |
|                     |                                            |
|                     v                                            |
|  3. Helm renders templates                                       |
|     substitutes {{ .Values.xxx }} with actual values             |
|                     |                                            |
|                     v                                            |
|  4. Helm sends the final YAML to the Kubernetes API server       |
|                     |                                            |
|                     v                                            |
|  5. Kubernetes creates all the objects                           |
|                     |                                            |
|                     v                                            |
|  6. Helm saves release metadata as a Secret in the cluster       |
|     (revision number, values used, rendered manifests)           |
|     This is what enables upgrade and rollback                    |
+------------------------------------------------------------------+
```

The release Secret is the key insight. Helm is not just a templating tool — it stores a complete snapshot of what was deployed and when.

---

## 3. Architecture

```
Your Machine                           Kubernetes Cluster
---------------------                  ---------------------------------

 +-----------+    helm install          +-------------------------+
 |           | -----------------------> |     API Server           |
 |  Helm CLI |                          |                         |
 |           | <----------------------- | +---------------------+ |
 +-----------+    status / output       | | Release Secret      | |
       |                                | | - rendered YAML      | |
       |                                | | - values used        | |
 +-----------+                          | | - revision: 3        | |
 |   Chart   |                          | +---------------------+ |
 | (local or |                          |                         |
 |   repo)   |                          | +---------------------+ |
 +-----------+                          | | Deployment          | |
                                        | | Service             | |
                                        | | ConfigMap           | |
                                        | +---------------------+ |
                                        +-------------------------+
```

Helm v3 has no server-side component. Tiller was removed in 2019 because it required a pod with cluster-admin permissions — a significant security risk. Helm v3 talks directly to the API server using your kubeconfig credentials.

---

## 4. Installation

```bash
# macOS
brew install helm

# Verify the install
helm version
# version.BuildInfo{Version:"v3.x.x", GitCommit:"...", ...}

# Helm uses whichever kubectl context is currently active
kubectl config current-context
kubectl get nodes   # if this works, Helm will work too
```

---

## 5. Repositories

A Helm repository is just an HTTP server hosting an `index.yaml` and packaged `.tgz` charts — conceptually identical to apt or npm registries.

```
Helm Repository (e.g. https://charts.bitnami.com/bitnami)
  |
  |-- index.yaml               <- list of all charts and versions
  |-- nginx-13.2.0.tgz         <- packaged chart
  |-- nginx-14.0.0.tgz
  |-- postgresql-12.1.2.tgz
  ...
```

```bash
# Add the repos you will use most
helm repo add bitnami       https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager  https://charts.jetstack.io
helm repo add prometheus    https://prometheus-community.github.io/helm-charts

# Pull latest chart metadata from all repos (like apt-get update)
helm repo update

# List your configured repos
helm repo list

# Search within your added repos
helm search repo nginx
helm search repo bitnami/postgres

# See every available version of a chart
helm search repo bitnami/nginx --versions

# Search Artifact Hub — the public registry for all community charts
helm search hub kafka

# Inspect a chart before installing
helm show chart  bitnami/nginx    # Chart.yaml
helm show values bitnami/nginx    # default values.yaml — read this before overriding
helm show readme bitnami/nginx    # full README
helm show all    bitnami/nginx    # everything above

# Remove a repo
helm repo remove bitnami
```

---

## 6. Charts — Deep Dive

### What is a chart?

A chart is a directory with a specific structure. When you run `helm package`, it becomes a `.tgz` file.

```bash
helm create my-app
```

This generates:

```
my-app/
|
|-- Chart.yaml           <- chart identity and metadata
|-- values.yaml          <- default configuration (the chart's public API)
|-- .helmignore          <- files to exclude from packaging (like .gitignore)
|
|-- charts/              <- downloaded sub-chart dependencies go here
|
+-- templates/           <- Go templates that render into K8s YAML
    |-- deployment.yaml
    |-- service.yaml
    |-- ingress.yaml
    |-- serviceaccount.yaml
    |-- hpa.yaml
    |-- _helpers.tpl     <- named templates shared across all files
    |-- NOTES.txt        <- text printed to the terminal after install
    +-- tests/
        +-- test-connection.yaml
```

---

### Chart.yaml

```yaml
# Chart.yaml — the chart's identity card

apiVersion: v2     # always v2 for Helm 3

name: my-app
description: A Node.js web application

type: application  # "application" (deployable) or "library" (just helpers, not deployable)

# chart version: increment this when you change the chart structure itself
# follows semver — 0.x.x for pre-release, 1.0.0+ for stable
version: 0.1.0

# appVersion: the version of the application inside the chart
# this is what shows up in "helm list" and is used as the default image tag
appVersion: "2.3.1"

# who maintains this chart
maintainers:
  - name: Prince Kumar
    email: prince@example.com
```

---

### values.yaml

This is the most important file in your chart. Think of it as the chart's public configuration API. Anyone deploying your chart will look here first.

```yaml
# values.yaml — defaults that users can override

# How many pod replicas to run
replicaCount: 1

# The container image to deploy
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""   # if empty, Chart.appVersion is used (see deployment.yaml template)

# Kubernetes Service configuration
service:
  type: ClusterIP
  port: 80

# Ingress — disabled by default, enable in your env-specific values file
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# CPU and memory allocation
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Horizontal Pod Autoscaler — disabled by default
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

# Additional environment variables to inject into the container
# Example:
# env:
#   - name: DB_HOST
#     value: "postgres-svc"
#   - name: LOG_LEVEL
#     value: "info"
env: []

nodeSelector: {}
tolerations: []
affinity: {}
```

---

### templates/deployment.yaml — Annotated

```yaml
# templates/deployment.yaml
# This is a normal K8s Deployment spec, but with {{ }} placeholders
# that Helm fills in from values.yaml (and any overrides).

apiVersion: apps/v1
kind: Deployment
metadata:
  # include "my-app.fullname" calls a named template from _helpers.tpl
  # it returns "releasename-chartname", max 63 chars (K8s DNS limit)
  name: {{ include "my-app.fullname" . }}
  labels:
    # another helper that adds the standard Helm-recommended labels
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  # Only set replicas if autoscaling is disabled.
  # If HPA is managing replicas, we must NOT set this field,
  # otherwise Helm would override the HPA on every upgrade.
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}

  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}

  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          # if image.tag is empty, fall back to Chart.AppVersion
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP

          # Only add the env block if values.env has at least one entry.
          # Without this guard, we would render an empty "env:" key which is valid
          # YAML but unnecessary noise in the manifest.
          {{- if .Values.env }}
          env:
            {{- toYaml .Values.env | nindent 12 }}
          {{- end }}

          resources:
            # toYaml converts the resources map to YAML.
            # nindent 12 adds the correct indentation.
            {{- toYaml .Values.resources | nindent 12 }}
```

---

### _helpers.tpl — Named Templates

```
{{/*
  my-app.fullname
  ---------------
  Returns the full name of the release: "releasename-chartname"
  Truncated to 63 characters because K8s DNS names have that limit.
  Users can override this entirely with .Values.fullnameOverride.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}


{{/*
  my-app.labels
  -------------
  Standard labels recommended by Helm.
  Applied to every resource so tools like kubectl, Helm, and monitoring
  systems can identify and group related objects.
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

---

### NOTES.txt — Post-Install Output

Helm prints this to the terminal after every install or upgrade. Use it to tell the user what was deployed and how to access it.

```
# templates/NOTES.txt
# This is a template too — {{ }} syntax works here.

Deployed: {{ .Chart.Name }} v{{ .Chart.AppVersion }}
Release:  {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}

To access the application:

{{- if eq .Values.service.type "NodePort" }}
  export NODE_PORT=$(kubectl get svc {{ include "my-app.fullname" . }} \
    -o jsonpath="{.spec.ports[0].nodePort}")
  export NODE_IP=$(kubectl get nodes \
    -o jsonpath="{.items[0].status.addresses[0].address}")
  echo "http://$NODE_IP:$NODE_PORT"
{{- else }}
  kubectl port-forward svc/{{ include "my-app.fullname" . }} 8080:{{ .Values.service.port }}
  # then visit: http://localhost:8080
{{- end }}
```

---

## 7. Values and Templating

### The template language

Helm uses Go's `text/template` package, extended with the Sprig function library.

```
{{ }}       evaluate a template expression and insert result
{{- }}      same, but strip all whitespace before this tag
{{- -}}     strip whitespace on both sides

.           the current context, called "the dot"
.Values     everything in values.yaml plus any overrides
.Release    information about this release
.Chart      contents of Chart.yaml
.Files      access to non-template files bundled in the chart
.Capabilities  Kubernetes version and available API groups
```

### Common patterns

```yaml
# Inject a value directly
replicas: {{ .Values.replicaCount }}

# Nested value
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}

# Default: if tag is empty string or nil, use appVersion
image: {{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}

# required: fail the install with a useful error if the value is missing
host: {{ required "ingress.host is required" .Values.ingress.host }}

# quote: wrap the value in double quotes (important for env var values)
value: {{ .Values.logLevel | quote }}

# upper / lower / title
env: {{ .Values.env | upper }}
```

### Conditionals

```yaml
# Basic if
{{- if .Values.ingress.enabled }}
# ingress YAML here
{{- end }}

# if not
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}

# if / else if / else
{{- if eq .Values.environment "production" }}
replicas: 5
{{- else if eq .Values.environment "staging" }}
replicas: 2
{{- else }}
replicas: 1
{{- end }}
```

### Loops

```yaml
# Loop over a list
# values.yaml:
#   env:
#     - name: DB_HOST
#       value: postgres
#     - name: LOG_LEVEL
#       value: debug

env:
{{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}


# Loop over a map — $key and $val are local variables
# values.yaml:
#   annotations:
#     prometheus.io/scrape: "true"
#     prometheus.io/port: "8080"

annotations:
{{- range $key, $val := .Values.annotations }}
  {{ $key }}: {{ $val | quote }}
{{- end }}
```

### toYaml with nindent

```yaml
# This is the most common pattern for blocks like resources, affinity, tolerations.
# toYaml serialises the map to YAML.
# nindent 12 adds a leading newline and 12 spaces of indentation to every line.

resources:
  {{- toYaml .Values.resources | nindent 12 }}

# With these values.yaml values:
# resources:
#   requests:
#     cpu: 100m
#     memory: 128Mi
#
# The output is:
# resources:
#             requests:
#               cpu: 100m
#               memory: 128Mi
```

### The .Release object

```yaml
# Built-in release information available in every template

name:      {{ .Release.Name }}        # the name you gave at install
namespace: {{ .Release.Namespace }}   # the namespace it was deployed into
revision:  {{ .Release.Revision }}    # 1 on first install, 2 after first upgrade, etc.
service:   {{ .Release.Service }}     # always "Helm"
isInstall: {{ .Release.IsInstall }}   # true only on the very first install
isUpgrade: {{ .Release.IsUpgrade }}   # true on every upgrade
```

### Value override priority

When you install or upgrade, Helm merges values from multiple sources. The rule is simple: later overrides earlier.

```
values.yaml (chart defaults)
     ^ overridden by
-f values-prod.yaml
     ^ overridden by
-f values-secrets.yaml
     ^ overridden by
--set image.tag=v2.0
```

---

## 8. Installing Releases

```bash
# Basic syntax: helm install <release-name> <chart>

# From a local directory
helm install my-app ./my-app

# From a repository
helm install my-nginx bitnami/nginx

# Specific version from a repo
helm install my-nginx bitnami/nginx --version 13.2.0

# Override values inline
helm install my-app ./my-app --set replicaCount=3

# Multiple --set flags
helm install my-app ./my-app --set replicaCount=3 --set image.tag=v2.0

# Override with a values file (preferred for anything more than one value)
helm install my-app ./my-app -f values-prod.yaml

# Merge multiple values files (prod wins over base defaults)
helm install my-app ./my-app -f values.yaml -f values-prod.yaml

# Deploy into a specific namespace, create it if it doesn't exist
helm install my-app ./my-app -n production --create-namespace

# Dry run: render the templates and validate against the cluster API, but do not create anything
helm install my-app ./my-app --dry-run --debug

# Wait until all pods are Running/Ready before returning success
helm install my-app ./my-app --wait --timeout 5m

# Automatically rollback if any pod fails to become ready within the timeout
helm install my-app ./my-app --atomic --timeout 5m
```

### Inspecting a release after install

```bash
# List all releases in the current namespace
helm list

# All releases in all namespaces
helm list -A

# Full status
helm status my-app

# What values are currently active (your overrides only)
helm get values my-app

# What values are active (overrides merged with defaults)
helm get values my-app --all

# The actual YAML Kubernetes received
helm get manifest my-app

# Everything Helm knows about this release
helm get all my-app
```

---

## 9. Upgrading Releases

```bash
# Upgrade with a new value
helm upgrade my-app ./my-app --set replicaCount=5

# Upgrade using a values file
helm upgrade my-app ./my-app -f values-prod.yaml

# upgrade --install: installs on first run, upgrades on subsequent runs.
# This is the standard pattern for CI/CD pipelines — one command that always works.
helm upgrade --install my-app ./my-app -f values-prod.yaml

# Roll back automatically if any pod fails to become healthy within the timeout.
# This is strongly recommended for production upgrades.
helm upgrade my-app ./my-app -f values-prod.yaml --atomic --wait --timeout 10m

# Keep only the last 10 revisions in history (saves storage in etcd)
helm upgrade my-app ./my-app --history-max 10

# Reuse the values from the previous release — only change what you specify
helm upgrade my-app ./my-app --reuse-values --set image.tag=v3.0

# Reset back to chart defaults, then apply your overrides
helm upgrade my-app ./my-app --reset-values -f values-prod.yaml
```

### --atomic explained

```
Without --atomic:
  Upgrade starts -> new pods crash -> Helm marks release as FAILED
  Your cluster is now in a broken state
  You have to manually run helm rollback

With --atomic:
  Upgrade starts -> new pods crash -> Helm automatically rolls back
  Release stays in a working state at all times
  You get a clear error: "UPGRADE FAILED: release has been rolled back"
```

---

## 10. Rollbacks

This is one of the biggest reasons to use Helm over raw kubectl. A rollback restores every Kubernetes resource to its previous state — not just the Deployment.

```
Release history:
  Revision 1  ->  app v1.0, nginx:1.23  (original install)
  Revision 2  ->  app v2.0, nginx:1.24  (first upgrade)
  Revision 3  ->  app v3.0, nginx:1.25  (second upgrade — broken)

helm rollback my-app 2  ->  restores revision 2 exactly
```

```bash
# Always check history before rolling back
helm history my-app

# Sample output:
# REVISION  UPDATED     STATUS      CHART        DESCRIPTION
# 1         10:00:00    superseded  my-app-0.1   Install complete
# 2         10:30:00    superseded  my-app-0.1   Upgrade complete
# 3         11:00:00    failed      my-app-0.1   Upgrade "my-app" failed
# 4         11:05:00    deployed    my-app-0.1   Rollback to 2

# Rollback to a specific revision
helm rollback my-app 2

# Rollback to the previous revision (one step back)
helm rollback my-app

# Rollback and wait for pods to be healthy
helm rollback my-app 2 --wait --timeout 5m
```

### Why helm rollback is better than kubectl rollout undo

```
kubectl rollout undo deployment/my-app
  - only rolls back the Deployment
  - ConfigMaps, Services, Secrets are NOT changed
  - if your upgrade changed a ConfigMap, kubectl rollout undo misses it

helm rollback my-app 2
  - restores the exact state of EVERY resource in the release
  - Deployment, Service, ConfigMap, Ingress, HPA — all of it
  - true atomic rollback
```

---

## 11. Hooks

Hooks let you run Kubernetes Jobs at specific points in a release's lifecycle — before or after install, upgrade, rollback, or uninstall.

```
Available hook points:

  pre-install    -> before any resources are created on first install
  post-install   -> after all resources are created
  pre-upgrade    -> before the upgrade begins
  post-upgrade   -> after the upgrade completes
  pre-rollback   -> before rollback begins
  post-rollback  -> after rollback completes
  pre-delete     -> before uninstall
  post-delete    -> after uninstall is done
  test           -> when you run "helm test"
```

### Example: run database migrations before deploying new pods

```yaml
# templates/pre-upgrade-migration.yaml

apiVersion: batch/v1
kind: Job
metadata:
  # include the revision number so each upgrade creates a uniquely named Job
  name: {{ include "my-app.fullname" . }}-migration-{{ .Release.Revision }}
  annotations:
    # tells Helm this is a hook, not a regular resource
    "helm.sh/hook": pre-upgrade

    # lower weight runs first — useful when you have multiple hooks
    "helm.sh/hook-weight": "-5"

    # delete the Job before creating the next one (avoids name conflicts on re-run)
    # also delete it after it succeeds (keep the cluster clean)
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3   # retry up to 3 times before marking the Job as failed
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migration
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["python", "manage.py", "migrate", "--noinput"]
          env:
            - name: DB_HOST
              value: {{ .Values.db.host | quote }}
```

### The upgrade flow with a hook

```
helm upgrade my-app ./my-app --set image.tag=v2.0
      |
      v
  1. pre-upgrade hook fires
     Kubernetes runs the migration Job
     Helm waits for it to complete successfully
      |
      v
  2. Rolling update begins
     New pods come up one at a time, old ones come down
      |
      v
  3. post-upgrade hook fires (optional — notification, smoke test, etc.)
      |
      v
  Done

If the migration Job fails:
  Helm marks the upgrade as FAILED
  (with --atomic it also rolls back automatically)
```

---

## 12. Chart Dependencies

Instead of writing your own PostgreSQL or Redis setup, declare them as dependencies and let Helm pull the battle-tested charts.

```yaml
# Chart.yaml — declare what your chart depends on

dependencies:
  - name: postgresql
    version: "12.1.2"
    repository: "https://charts.bitnami.com/bitnami"
    # only install this sub-chart if postgresql.enabled is true in values
    condition: postgresql.enabled

  - name: redis
    version: "17.3.11"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
# Downloads sub-charts into charts/ directory and generates Chart.lock
helm dependency update ./my-app

# Verify dependency state
helm dependency list ./my-app
```

```yaml
# values.yaml — configure the sub-charts alongside your own app

# Your app config
image:
  repository: my-registry/my-app
  tag: "1.0.0"

# PostgreSQL sub-chart config
# Keys under "postgresql:" pass directly to the postgresql chart's values
postgresql:
  enabled: true
  auth:
    username: myuser
    password: "changeme"   # use a Secret or helm-secrets in real deployments
    database: mydb
  primary:
    persistence:
      enabled: true
      size: 20Gi

# Redis sub-chart config
redis:
  enabled: true
  auth:
    enabled: false         # no password, internal access only
  master:
    persistence:
      enabled: false       # stateless cache, no persistence needed
```

```bash
# One command deploys your app + postgres + redis
helm install my-app ./my-app -f values.yaml -n dev --create-namespace
```

### Disabling sub-charts per environment

```yaml
# values-prod.yaml
# In production, postgres is AWS RDS — no need to deploy it in the cluster
postgresql:
  enabled: false

# Point the app at the external database
env:
  - name: DATABASE_URL
    value: "postgresql://myuser@prod-db.us-east-1.rds.amazonaws.com:5432/mydb"

redis:
  enabled: true
  replica:
    replicaCount: 2
```

---

## 13. Packaging and Sharing

```bash
# Package a chart directory into a versioned .tgz file
helm package ./my-app
# produces: my-app-0.1.0.tgz

# Package with a specific destination directory
helm package ./my-app -d ./releases/

# Install directly from a .tgz
helm install my-release ./my-app-0.1.0.tgz

# Generate an index.yaml for hosting your own chart repository
helm repo index ./releases/ --url https://charts.your-domain.com

# Others can then add your repo and install from it
helm repo add yourrepo https://charts.your-domain.com
helm install my-release yourrepo/my-app

# Push to an OCI registry (Docker Hub, ECR, GCR) — Helm 3.8+
helm push my-app-0.1.0.tgz oci://registry.example.com/charts
helm install my-release oci://registry.example.com/charts/my-app --version 0.1.0
```

---

## 14. Testing Charts

### helm template — see what Helm will actually send to Kubernetes

```bash
# Render all templates locally — no cluster needed
helm template my-release ./my-app

# Render with specific overrides
helm template my-release ./my-app -f values-prod.yaml --set image.tag=v2.0

# Render only one specific template file
helm template my-release ./my-app -s templates/deployment.yaml

# Pipe rendered output to kubectl dry-run for extra validation
helm template my-release ./my-app | kubectl apply --dry-run=client -f -

# Save to file and inspect
helm template my-release ./my-app > /tmp/rendered.yaml
cat /tmp/rendered.yaml
```

### helm lint — validate chart structure

```bash
helm lint ./my-app                      # check for errors and warnings
helm lint ./my-app -f values-prod.yaml  # lint with specific values
helm lint ./my-app --strict             # treat warnings as errors
```

### helm test — run test pods against a live release

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "my-app.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test   # this pod only runs when "helm test" is called
spec:
  containers:
    - name: test
      image: busybox
      # test that the service is reachable and returns HTTP 200
      command: ['wget', '--spider', '-q',
        'http://{{ include "my-app.fullname" . }}:{{ .Values.service.port }}/health']
  restartPolicy: Never
```

```bash
helm install my-release ./my-app
helm test my-release          # runs the test pod
helm test my-release --logs   # shows the test pod's output
```

---

## 15. Handling Secrets

### The problem

Never put real credentials in `values.yaml` and commit that to git.

### Option 1: pass via environment variable at deploy time

```bash
# The CI/CD system injects secrets from its own secret store
helm upgrade --install my-app ./my-app \
  -f values-prod.yaml \
  --set db.password="${DB_PASSWORD}"   # read from CI/CD env var, not hardcoded
```

### Option 2: separate secrets file, gitignored

```bash
# values-secrets.yaml — this file is in .gitignore
db:
  password: "actual-production-password"
  apiKey:   "actual-api-key"
```

```bash
# .gitignore
values-secrets.yaml
```

```bash
# Deploy with both files — secrets override on top
helm upgrade --install my-app ./my-app \
  -f values-prod.yaml \
  -f values-secrets.yaml
```

### Option 3: helm-secrets plugin (recommended for teams)

```bash
# Install the plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Encrypt the secrets file using sops (with age or GPG key)
helm secrets encrypt values-secrets.yaml
# produces values-secrets.yaml.enc — safe to commit

# Deploy: helm-secrets decrypts transparently at runtime
helm secrets upgrade --install my-app ./my-app \
  -f values-prod.yaml \
  -f values-secrets.yaml.enc
```

---

## 16. Things Worth Knowing

**Helm was created in 2015 at Deis** (acquired by Microsoft). It was donated to the CNCF and is now one of the most widely adopted projects in the cloud-native ecosystem.

**The name fits the Kubernetes nautical theme.** Kubernetes is Greek for "helmsman." Helm is the steering wheel. The ship analogy extends to releases (voyages) and charts (navigation maps).

**Helm v2 required a server-side component called Tiller.** Tiller ran in the cluster with cluster-admin privileges, which was a serious security problem. Helm v3 (released November 2019) removed it entirely. The security model is now identical to kubectl.

**Each Helm release is stored as a Kubernetes Secret** in the namespace it was deployed to. You can see them with `kubectl get secrets | grep helm`. Each revision is a separate compressed, base64-encoded Secret.

**`helm upgrade --install` is the standard CI/CD command** because it is idempotent — it works whether the release exists or not. You never need two separate pipeline steps for "first deploy" vs "update."

**Rollback creates a new revision, not a revert.** After `helm rollback my-app 2`, the history shows a new revision 4 (or whatever comes next) with description "Rollback to 2." The old revisions are preserved.

**Artifact Hub (artifacthub.io) hosts over 10,000 Helm charts.** Before writing a chart for a common dependency like PostgreSQL, Redis, or RabbitMQ, check there first.

---

> Next: [03_HELM_CHEATSHEET.md](./03_HELM_CHEATSHEET.md)
