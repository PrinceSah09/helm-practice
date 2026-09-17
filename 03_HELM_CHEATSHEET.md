# Helm Cheatsheet

All commands grouped by what you are trying to do.
Every flag includes a comment explaining when to use it.

---

## Table of Contents

1. [Setup](#1-setup)
2. [Repositories](#2-repositories)
3. [Search and Inspect](#3-search-and-inspect)
4. [Install](#4-install)
5. [Upgrade](#5-upgrade)
6. [Rollback](#6-rollback)
7. [Uninstall](#7-uninstall)
8. [Inspect a Release](#8-inspect-a-release)
9. [Chart Development](#9-chart-development)
10. [Templating Reference](#10-templating-reference)
11. [Dependencies](#11-dependencies)
12. [Packaging and Publishing](#12-packaging-and-publishing)
13. [Plugins](#13-plugins)
14. [Flags Table](#14-flags-table)
15. [Template Functions](#15-template-functions)
16. [Values Override Patterns](#16-values-override-patterns)
17. [CI/CD Patterns](#17-cicd-patterns)
18. [Quick Reference Card](#18-quick-reference-card)

---

## 1. Setup

```bash
# Install on macOS
brew install helm

# Confirm the version
helm version

# See Helm's configuration (cache path, data path, plugins, etc.)
helm env

# Helm uses whatever kubectl context is currently active
kubectl config current-context          # see active context
kubectl config get-contexts             # list all contexts
kubectl config use-context <name>       # switch context

# Helm commands default to the current namespace unless you pass -n
kubectl config set-context --current --namespace=production
```

---

## 2. Repositories

```bash
# -- ADDING REPOS -----------------------------------------------------------
# Common ones you will use regularly
helm repo add bitnami       https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager  https://charts.jetstack.io
helm repo add prometheus    https://prometheus-community.github.io/helm-charts
helm repo add grafana       https://grafana.github.io/helm-charts
helm repo add argo          https://argoproj.github.io/argo-helm

# -- MAINTAINING REPOS ------------------------------------------------------
helm repo list                          # what repos you have configured
helm repo update                        # pull latest chart metadata (do this before searching)
helm repo update bitnami                # update one specific repo
helm repo remove bitnami                # remove a repo

# -- HOSTING YOUR OWN -------------------------------------------------------
# Generate index.yaml for a directory of .tgz chart files
helm repo index ./charts/ --url https://charts.your-domain.com
# Upload the directory to any static HTTP server — that's your repo
```

---

## 3. Search and Inspect

```bash
# -- SEARCH -----------------------------------------------------------------
helm search repo nginx                  # search in your configured repos
helm search repo bitnami/postgres       # narrow to a specific repo
helm search repo bitnami/nginx --versions  # show every available version

helm search hub wordpress               # search Artifact Hub (artifacthub.io)
helm search hub kafka

# -- INSPECT BEFORE INSTALLING ----------------------------------------------
# Always read the default values before overriding them
helm show chart  bitnami/nginx          # Chart.yaml metadata
helm show values bitnami/nginx          # full default values.yaml — read this first
helm show readme bitnami/nginx          # README
helm show all    bitnami/nginx          # everything above in one shot

# Show a specific version
helm show values bitnami/nginx --version 13.2.0
```

---

## 4. Install

```bash
# -- BASIC ------------------------------------------------------------------
helm install <release-name> <chart>

# From a local directory
helm install my-app ./my-app

# From a repo
helm install my-nginx bitnami/nginx

# Pin to a specific chart version
helm install my-nginx bitnami/nginx --version 13.2.0

# -- NAMESPACE --------------------------------------------------------------
helm install my-app ./my-app -n production
helm install my-app ./my-app -n production --create-namespace  # create ns if missing

# -- OVERRIDING VALUES ------------------------------------------------------
# Single value
helm install my-app ./my-app --set replicaCount=3

# Multiple values inline
helm install my-app ./my-app --set replicaCount=3 --set image.tag=v2.0

# From a values file (preferred over --set for anything complex)
helm install my-app ./my-app -f values-prod.yaml

# Merge multiple files: base defaults + environment overrides
helm install my-app ./my-app -f values.yaml -f values-prod.yaml

# -- SAFETY FLAGS -----------------------------------------------------------
# Dry run: render templates and validate against the API, nothing is created
helm install my-app ./my-app --dry-run

# Dry run + verbose template output for debugging
helm install my-app ./my-app --dry-run --debug

# Wait until all pods reach Running/Ready before returning success
helm install my-app ./my-app --wait

# Same as --wait, but rolls back automatically if pods do not become healthy
helm install my-app ./my-app --atomic --timeout 5m

# -- OTHER ------------------------------------------------------------------
# Let Helm generate a random release name
helm install ./my-app --generate-name

# Force values to be treated as strings, not parsed (e.g. "1.25" stays a string)
helm install my-app ./my-app --set-string image.tag="1.25"

# Read a value from a local file
helm install my-app ./my-app --set-file nginxConfig=./nginx.conf

# Attach a description visible in helm history
helm install my-app ./my-app --description "initial production deploy"
```

---

## 5. Upgrade

```bash
# -- BASIC ------------------------------------------------------------------
helm upgrade my-app ./my-app --set replicaCount=5
helm upgrade my-app ./my-app -f values-prod.yaml

# Upgrade to a newer chart version from a repo
helm upgrade my-nginx bitnami/nginx
helm upgrade my-nginx bitnami/nginx --version 14.0.0

# -- upgrade --install: the standard CI/CD pattern -------------------------
# Installs if the release does not exist, upgrades if it does.
# Use this everywhere in CI/CD — one command, always correct.
helm upgrade --install my-app ./my-app -f values-prod.yaml -n prod --create-namespace

# -- SAFETY FLAGS -----------------------------------------------------------
# Wait for pods before returning
helm upgrade my-app ./my-app --wait --timeout 10m

# Roll back automatically if pods fail to become healthy within the timeout
# Use this in production — it prevents leaving the cluster in a broken state
helm upgrade my-app ./my-app --atomic --wait --timeout 10m

# Remove any newly created resources if the upgrade fails
helm upgrade my-app ./my-app --cleanup-on-fail

# -- VALUE MANAGEMENT -------------------------------------------------------
# Keep all values from the previous release, only change what you specify
helm upgrade my-app ./my-app --reuse-values --set image.tag=v3.0

# Discard previous overrides, start from chart defaults + your new -f file
helm upgrade my-app ./my-app --reset-values -f values-prod.yaml

# -- HISTORY MANAGEMENT -----------------------------------------------------
# Keep only the last 10 revisions (each revision is a Secret in etcd)
helm upgrade my-app ./my-app --history-max 10

# -- FORCE ------------------------------------------------------------------
# Delete and recreate resources that cannot be patched (use with care)
helm upgrade my-app ./my-app --force
```

---

## 6. Rollback

```bash
# -- ALWAYS CHECK HISTORY FIRST ---------------------------------------------
helm history my-app

# Example output:
# REVISION  UPDATED       STATUS      CHART        APP VER  DESCRIPTION
# 1         Mon 10:00     superseded  my-app-0.1   1.0.0    Install complete
# 2         Mon 10:30     superseded  my-app-0.1   2.0.0    Upgrade complete
# 3         Mon 11:00     failed      my-app-0.1   3.0.0    Upgrade failed
# 4         Mon 11:05     deployed    my-app-0.1   2.0.0    Rollback to 2

# -- ROLLBACK ---------------------------------------------------------------
helm rollback my-app 2              # rollback to a specific revision
helm rollback my-app                # rollback to the previous revision

# Wait for pods to be healthy before returning success
helm rollback my-app 2 --wait --timeout 5m

# Delete resources that were added during the failed upgrade
helm rollback my-app 2 --cleanup-on-fail

# -- VERIFY AFTER ROLLBACK --------------------------------------------------
helm status my-app                  # confirm STATUS is "deployed"
helm history my-app                 # a new revision is appended, not a revert

# -- HISTORY OPTIONS --------------------------------------------------------
helm history my-app --max 5         # show only the last 5 revisions
helm history my-app -o json         # machine-readable output
```

---

## 7. Uninstall

```bash
# Remove a release and all its resources
helm uninstall my-app

# Remove from a specific namespace
helm uninstall my-app -n production

# Remove but keep the history (allows rollback even after uninstall)
helm uninstall my-app --keep-history

# See what would be deleted without actually deleting
helm uninstall my-app --dry-run
```

---

## 8. Inspect a Release

```bash
# -- LIST -------------------------------------------------------------------
helm list                           # current namespace
helm list -A                        # all namespaces
helm list -n production             # specific namespace
helm list --failed                  # only releases in FAILED state
helm list --deployed                # only deployed releases
helm list -o json                   # JSON output (useful for scripting)

# -- STATUS -----------------------------------------------------------------
helm status my-app                  # release status + NOTES.txt output
helm status my-app -o json          # JSON format

# -- GET --------------------------------------------------------------------
# What values are currently active (your overrides only, not defaults)
helm get values my-app

# Merged values: your overrides + chart defaults
helm get values my-app --all

# Values that were in effect at a specific revision
helm get values my-app --revision 2

# The exact YAML Kubernetes received at last deploy
helm get manifest my-app

# The YAML that was applied at a specific revision
helm get manifest my-app --revision 2

# Hooks defined in this release
helm get hooks my-app

# NOTES.txt output from the last deploy
helm get notes my-app

# Everything: values + manifest + hooks + notes
helm get all my-app
```

---

## 9. Chart Development

```bash
# -- CREATE -----------------------------------------------------------------
helm create my-app          # scaffold a new chart with sensible defaults

# -- VALIDATE ---------------------------------------------------------------
helm lint ./my-app                          # check for errors and warnings
helm lint ./my-app -f values-prod.yaml      # lint with specific values
helm lint ./my-app --strict                 # fail on warnings too

# -- RENDER TEMPLATES -------------------------------------------------------
# See the YAML that would be applied, without touching the cluster
helm template my-release ./my-app

# Render with value overrides
helm template my-release ./my-app -f values-prod.yaml

# Render only one specific template
helm template my-release ./my-app -s templates/deployment.yaml

# Pipe to kubectl dry-run for K8s-level validation
helm template my-release ./my-app | kubectl apply --dry-run=client -f -

# Save rendered output to a file
helm template my-release ./my-app > /tmp/rendered.yaml

# -- TEST -------------------------------------------------------------------
# Runs test hooks (pods annotated with helm.sh/hook: test)
helm test my-release
helm test my-release --logs             # also show test pod stdout
```

---

## 10. Templating Reference

### The dot context

```
.Values         -> everything from values.yaml + your overrides
.Release        -> release metadata
  .Release.Name         "my-app"
  .Release.Namespace    "production"
  .Release.Revision     3
  .Release.IsInstall    true on first install, false on upgrades
  .Release.IsUpgrade    true on upgrades, false on first install
.Chart          -> Chart.yaml contents
  .Chart.Name           "my-app"
  .Chart.Version        "0.1.0"
  .Chart.AppVersion     "2.3.1"
.Files          -> access to non-template files in the chart
.Capabilities   -> cluster API information
  .Capabilities.KubeVersion.Major    "1"
  .Capabilities.KubeVersion.Minor    "28"
```

### Template syntax

```
{{ .Values.key }}                  inject value
{{- .Values.key }}                 inject value, trim preceding whitespace
{{- .Values.key -}}                inject value, trim whitespace both sides

{{ .Values.key | default "x" }}    use "x" if key is empty or nil
{{ .Values.key | quote }}          wrap in double quotes: "value"
{{ .Values.key | upper }}          UPPERCASE
{{ .Values.key | lower }}          lowercase
{{ .Values.key | trunc 63 }}       truncate to 63 characters
{{ .Values.num  | int }}           convert to integer
{{ .Values.list | toYaml | nindent 4 }}   serialize to YAML with indentation
{{ include "chart.helper" . }}     call a named template from _helpers.tpl
{{ required "must set X" .Values.x }}    fail install if .Values.x is missing
```

### Control flow

```
{{- if .Values.ingress.enabled }}
# yaml here
{{- end }}

{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}

{{- if eq .Values.env "production" }}
replicas: 5
{{- else if eq .Values.env "staging" }}
replicas: 2
{{- else }}
replicas: 1
{{- end }}

# Range over a list
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}

# Range over a map ($key and $val are local variables)
{{- range $key, $val := .Values.labels }}
  {{ $key }}: {{ $val | quote }}
{{- end }}

# with: sets . to a sub-value (also acts as an nil guard)
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

---

## 11. Dependencies

```bash
# Download declared dependencies into charts/
helm dependency update ./my-app

# See dependency status (missing, ok, wrong version, etc.)
helm dependency list ./my-app

# Build from Chart.lock without pulling new versions
helm dependency build ./my-app
```

```yaml
# Chart.yaml — dependency declarations

dependencies:
  - name: postgresql
    version: "12.1.2"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled   # controlled by values.postgresql.enabled

  - name: redis
    version: "17.x.x"              # x = accept any patch version
    repository: "https://charts.bitnami.com/bitnami"
    alias: cache                    # reference it as "cache" in values.yaml
    tags:
      - backend                     # enable/disable all "backend"-tagged deps together
```

---

## 12. Packaging and Publishing

```bash
# Package a chart directory into a versioned .tgz
helm package ./my-app
# output: my-app-0.1.0.tgz

# Specify the output directory
helm package ./my-app -d ./releases/

# Override the version for this package
helm package ./my-app --version 1.2.3

# Install directly from a .tgz (no repo needed)
helm install my-release ./my-app-0.1.0.tgz

# Push to OCI registry (Helm 3.8+)
helm push my-app-0.1.0.tgz oci://registry.example.com/charts
helm install my-release oci://registry.example.com/charts/my-app --version 0.1.0

# Generate index.yaml for a static file host
helm repo index ./releases/ --url https://charts.your-domain.com
```

---

## 13. Plugins

```bash
helm plugin list
helm plugin install <url>
helm plugin update <name>
helm plugin remove <name>
```

### helm-diff — see exactly what will change before upgrading

```bash
helm plugin install https://github.com/databus23/helm-diff

# Compare current release with what the upgrade would produce
helm diff upgrade my-app ./my-app -f values-prod.yaml

# + lines are being added
# - lines are being removed
# Run this before every production upgrade
```

### helm-secrets — encrypt sensitive values with sops

```bash
helm plugin install https://github.com/jkroepke/helm-secrets

# Encrypt a values file (uses age or GPG under the hood)
helm secrets encrypt values-secrets.yaml

# Deploy with automatic decryption at runtime
helm secrets upgrade --install my-app ./my-app \
  -f values-prod.yaml \
  -f values-secrets.yaml.enc
```

---

## 14. Flags Table

| Flag | Commands | What it does |
|------|----------|--------------|
| `--set key=val` | install, upgrade | Override a single value inline |
| `--set-string key=val` | install, upgrade | Override as a string — prevents type coercion |
| `--set-file key=./file` | install, upgrade | Read value from a local file |
| `-f file.yaml` | install, upgrade | Override values from a file |
| `--values file.yaml` | install, upgrade | Same as `-f` |
| `--reuse-values` | upgrade | Keep all values from previous release |
| `--reset-values` | upgrade | Discard previous values, start from chart defaults |
| `--dry-run` | install, upgrade, uninstall | Simulate without applying anything |
| `--debug` | any | Print verbose output including rendered templates |
| `--wait` | install, upgrade, rollback | Block until pods are Running/Ready |
| `--timeout 5m` | install, upgrade, rollback | How long to wait (default 5m) |
| `--atomic` | install, upgrade | Roll back automatically if pods fail |
| `--cleanup-on-fail` | upgrade, rollback | Delete new resources if the operation fails |
| `--force` | upgrade | Delete and recreate resources that cannot be patched |
| `--history-max N` | upgrade | Maximum revisions to keep in history |
| `-n namespace` | any | Target namespace |
| `--create-namespace` | install | Create the namespace if it does not exist |
| `--generate-name` | install | Auto-generate the release name |
| `--description "text"` | install, upgrade | Custom description visible in `helm history` |
| `--version x.y.z` | install, upgrade | Use a specific chart version |
| `-o json/yaml/table` | list, status, history | Output format |
| `--keep-history` | uninstall | Preserve history after removing the release |
| `--all` | list | Include failed and uninstalled releases |
| `-A` | list | All namespaces |
| `--max N` | history | Show only the N most recent revisions |
| `-s templates/x.yaml` | template | Render only that specific template file |
| `--strict` | lint | Treat warnings as errors |

---

## 15. Template Functions

```
STRING
------------------------------------------------------------------
upper                          "hello"        -> "HELLO"
lower                          "HELLO"        -> "hello"
title                          "hello world"  -> "Hello World"
trim                           "  hello  "    -> "hello"
trimSuffix "-" "my-app-"       -> "my-app"
trimPrefix "my-" "my-app"      -> "app"
replace "app" "svc" "my-app"   -> "my-svc"
quote                          hello          -> "hello"
squote                         hello          -> 'hello'
printf "%s-%s" .a .b           -> "a-b"
contains "bc" "abcde"          -> true
hasPrefix "my" "my-app"        -> true
hasSuffix "app" "my-app"       -> true
trunc 63                       -> truncate to 63 chars
nospace                        -> remove all whitespace
indent N                       -> add N spaces to each line
nindent N                      -> add newline + N spaces to each line
regexMatch "^[a-z]+" .name     -> true/false

NUMBER
------------------------------------------------------------------
add 1 2        -> 3
sub 5 2        -> 3
mul 2 3        -> 6
div 10 2       -> 5
mod 10 3       -> 1
max 3 5        -> 5
min 3 5        -> 3
ceil 1.5       -> 2
floor 1.9      -> 1
int "5"        -> 5

LIST
------------------------------------------------------------------
list 1 2 3          -> [1 2 3]
first (list 1 2 3)  -> 1
last  (list 1 2 3)  -> 3
len   (list 1 2 3)  -> 3
append (list 1 2) 3 -> [1 2 3]
has 2 (list 1 2 3)  -> true
uniq  (list 1 1 2)  -> [1 2]
reverse (list 1 2)  -> [2 1]

DICT
------------------------------------------------------------------
dict "key" "val"             -> {key: val}
set    $d "key" "val"        -> mutate $d
unset  $d "key"              -> remove key
hasKey $d "key"              -> true/false
keys   $d                    -> list of keys
values $d                    -> list of values
merge  $d1 $d2               -> $d2 wins on conflict

TYPE CONVERSION
------------------------------------------------------------------
toString    -> convert to string
toJson      -> serialize to JSON string
fromJson    -> parse JSON string
toYaml      -> serialize to YAML string
fromYaml    -> parse YAML string
b64enc      -> base64 encode
b64dec      -> base64 decode

RANDOM / CRYPTO
------------------------------------------------------------------
randAlphaNum 10    -> "aB3xKp2mNq"     (letters + digits)
randAlpha 8        -> "AbCdEfGh"       (letters only)
randNumeric 6      -> "847291"         (digits only)
uuidv4             -> "b02d4a8e-..."
sha256sum "data"   -> sha256 hash string
```

---

## 16. Values Override Patterns

### Pattern 1 — Environment-specific values files

```bash
# One base file with shared defaults, one per environment with only the differences

helm upgrade --install my-app ./my-app \
  -f values.yaml \
  -f values-dev.yaml \
  -n dev

helm upgrade --install my-app ./my-app \
  -f values.yaml \
  -f values-staging.yaml \
  -n staging

helm upgrade --install my-app ./my-app \
  -f values.yaml \
  -f values-prod.yaml \
  -n prod
```

### Pattern 2 — Image tag from CI/CD

```bash
# The image tag is the only thing that changes on each deploy.
# Pass it via --set; everything else comes from the values file.

IMAGE_TAG=$(git rev-parse --short HEAD)   # e.g., "a1b2c3d"

helm upgrade --install my-app ./my-app \
  -f values-prod.yaml \
  --set image.tag=${IMAGE_TAG} \
  --atomic \
  --wait
```

### Pattern 3 — Nested values

```bash
# Use dot notation to reach nested keys
helm install my-app ./my-app \
  --set image.repository=my-registry/my-app \
  --set image.tag=v2.0 \
  --set service.type=LoadBalancer \
  --set ingress.enabled=true \
  --set "ingress.hosts[0].host=myapp.example.com"
```

### Pattern 4 — List values via --set

```bash
# Set individual items in a list
helm install my-app ./my-app \
  --set "env[0].name=DB_HOST"    \
  --set "env[0].value=postgres"  \
  --set "env[1].name=LOG_LEVEL"  \
  --set "env[1].value=debug"
```

### Pattern 5 — Escaping special characters

```bash
# Commas and equals signs are special in --set, escape them with backslash
helm install my-app ./my-app \
  --set "labels.value=first\,second"   # comma in a value
  --set "config.key=one\=two"          # equals sign in a value
```

---

## 17. CI/CD Patterns

### Full deploy step in GitHub Actions

```yaml
# .github/workflows/deploy.yml

- name: Deploy to Kubernetes
  run: |
    helm upgrade --install ${{ env.RELEASE_NAME }} ./helm/my-app \
      --namespace ${{ env.NAMESPACE }}              \
      --create-namespace                            \
      -f helm/my-app/values.yaml                   \
      -f helm/my-app/values-${{ env.ENV }}.yaml    \
      --set image.tag=${{ github.sha }}             \
      --wait                                        \
      --atomic                                      \
      --timeout 10m                                 \
      --history-max 10
```

### Idempotent deploy script (runs on first deploy and every update)

```bash
#!/usr/bin/env bash
set -euo pipefail

RELEASE=my-app
NAMESPACE=production
IMAGE_TAG=$(git rev-parse --short HEAD)

helm upgrade --install "${RELEASE}" ./my-app      \
  -f values-prod.yaml                             \
  --set image.tag="${IMAGE_TAG}"                  \
  -n "${NAMESPACE}"                               \
  --create-namespace                              \
  --atomic                                        \
  --wait                                          \
  --timeout 10m                                   \
  --history-max 20
```

### Check status in a script

```bash
# Exit 0 if deployed, non-zero if not found or failed
helm status my-app -n production > /dev/null 2>&1 && echo "deployed" || echo "not found"
```

---

## 18. Quick Reference Card

```
+------------------+------------------------------------------+
|  INSTALL         |  helm install <name> <chart>             |
|  UPGRADE         |  helm upgrade <name> <chart>             |
|  INSTALL/UPGRADE |  helm upgrade --install <name> <chart>   |
|  ROLLBACK        |  helm rollback <name> [revision]         |
|  UNINSTALL       |  helm uninstall <name>                   |
+------------------+------------------------------------------+
|  LIST            |  helm list -A                            |
|  STATUS          |  helm status <name>                      |
|  HISTORY         |  helm history <name>                     |
|  VALUES (active) |  helm get values <name>                  |
|  VALUES (all)    |  helm get values <name> --all            |
|  MANIFEST        |  helm get manifest <name>                |
+------------------+------------------------------------------+
|  ADD REPO        |  helm repo add <name> <url>              |
|  UPDATE REPOS    |  helm repo update                        |
|  SEARCH          |  helm search repo <term>                 |
|  SHOW VALUES     |  helm show values <chart>                |
+------------------+------------------------------------------+
|  CREATE CHART    |  helm create <name>                      |
|  RENDER          |  helm template <name> <chart>            |
|  LINT            |  helm lint <chart>                       |
|  TEST            |  helm test <name>                        |
|  PACKAGE         |  helm package <chart>                    |
+------------------+------------------------------------------+
```

---

> Next: [04_REAL_WORLD_PROBLEMS.md](./04_REAL_WORLD_PROBLEMS.md)
