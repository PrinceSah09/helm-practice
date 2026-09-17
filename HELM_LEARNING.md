# Helm + Kubernetes — Learning Guide

This guide is structured for someone who already has KIND running and wants a solid, hands-on understanding of both Kubernetes and Helm. No fluff.

---

## Files in This Guide

```
helm practice/
├── HELM_LEARNING.md               <- start here
├── 01_KUBERNETES_FUNDAMENTALS.md  <- K8s concepts, architecture, kubectl
├── 02_HELM_COMPLETE_GUIDE.md      <- everything about Helm, from scratch
├── 03_HELM_CHEATSHEET.md          <- every command and flag, grouped
└── 04_REAL_WORLD_PROBLEMS.md      <- 5 real problems solved with Helm
```

---

## Where to Start

```
Never touched K8s before?
  → 01_KUBERNETES_FUNDAMENTALS.md first, then come back

Know K8s, new to Helm?
  → jump straight to 02_HELM_COMPLETE_GUIDE.md

Know Helm, need a reference?
  → 03_HELM_CHEATSHEET.md

Want to see it used on real problems?
  → 04_REAL_WORLD_PROBLEMS.md
```

---

## What Each File Covers

### [01 — Kubernetes Fundamentals](./01_KUBERNETES_FUNDAMENTALS.md)

| Topic | What you will learn |
|-------|-------------------|
| Why K8s exists | The problem, from the ground up |
| Cluster architecture | Control plane, worker nodes, each component's job |
| Pod | What it is, why it's ephemeral, how to work with it |
| Deployment | Self-healing, rolling updates, replica management |
| Service | ClusterIP, NodePort, LoadBalancer, how DNS works |
| ConfigMap and Secret | Externalising config from your container image |
| Namespace | Isolating environments inside one cluster |
| HPA, PV, Jobs | Autoscaling, persistent storage, batch workloads |
| Networking | Pod-to-pod communication, DNS, full request flow |
| RBAC | Role-based access control |
| kubectl reference | Commands grouped by what you are trying to do |

---

### [02 — Helm Complete Guide](./02_HELM_COMPLETE_GUIDE.md)

| Topic | What you will learn |
|-------|-------------------|
| Why Helm exists | The specific pain of managing raw YAML at scale |
| Architecture | How Helm works internally, why Tiller is gone in v3 |
| Repositories | Adding, searching, and updating chart repos |
| Chart anatomy | Chart.yaml, values.yaml, templates/, _helpers.tpl, NOTES.txt |
| Templating | Go template syntax, every pattern you will use |
| Values | Overriding, merging, priority order |
| Install | All flags with explanations |
| Upgrade | --atomic, --reuse-values, --force and when to use each |
| Rollback | What actually gets rolled back vs kubectl rollout undo |
| Hooks | pre-upgrade, post-install, running DB migrations |
| Dependencies | Sub-charts, conditions, composing a full stack |
| Packaging | Creating .tgz, hosting a chart repository |
| Testing | helm template, helm lint, helm test |
| Secrets | Handling sensitive values safely |

---

### [03 — Helm Cheatsheet](./03_HELM_CHEATSHEET.md)

| Section | Contents |
|---------|----------|
| Setup | Install Helm, check context |
| Repos | add, list, update, remove, index |
| Search | search repo, search hub, show all |
| Install | every install flag with comments |
| Upgrade | --install, --atomic, --reuse-values |
| Rollback | history output, rollback variants |
| Uninstall | --keep-history |
| Inspect | list, status, get values / manifest / hooks |
| Chart development | create, lint, template, test, package |
| Flags table | every flag, what it does, when to use it |
| Template functions | string, number, list, dict, conversion |
| Values override patterns | env files, CI/CD, nested, lists, escaping |
| CI/CD | GitHub Actions, idempotent deploy pattern |
| Quick reference card | all commands in one block |

---

### [04 — Real World Problems](./04_REAL_WORLD_PROBLEMS.md)

| Problem | What it covers |
|---------|----------------|
| Multi-environment deploy | One chart, three environments, values file merging |
| Zero-downtime upgrade | Hooks, rolling update, --atomic, readiness probes |
| Generic microservice chart | One chart for eight services, per-service values files |
| Production rollback | Full rollback flow, what helm rollback actually restores |
| App + PostgreSQL + Redis | Chart dependencies, composing a stack, migrations |

---

## Quick Start — Run This Now

```bash
# Confirm your KIND cluster is up
kubectl get nodes

# Install Helm
brew install helm

# Add the Bitnami chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install nginx — your first Helm release
helm install my-nginx bitnami/nginx

# See what was created in the cluster
kubectl get all

# See the Helm release
helm list
helm status my-nginx

# Upgrade it — change replica count
helm upgrade my-nginx bitnami/nginx --set replicaCount=2

# See the revision history
helm history my-nginx

# Roll back to the original install
helm rollback my-nginx 1
helm history my-nginx

# Clean up
helm uninstall my-nginx

# Now create your own chart from scratch
helm create my-app
helm install my-release ./my-app
kubectl get pods
```

---

## Key Mental Models

```
Kubernetes is the operating system for containers.
Helm is the package manager for Kubernetes.

apt install nginx         =  helm install my-nginx bitnami/nginx
apt upgrade nginx         =  helm upgrade my-nginx bitnami/nginx
apt remove nginx          =  helm uninstall my-nginx
dpkg --list               =  helm list

# apt cannot do this. Helm can.
apt rollback nginx        =  helm rollback my-nginx 1
```

---

```
A Helm release stores three things:
  1. the rendered YAML that was applied to the cluster
  2. the values that were used to render it
  3. a revision number

This is what makes upgrade and rollback possible.
Without this metadata, Helm would be just a template renderer.
```

---

```
The three files you will spend most of your time in:

  Chart.yaml   -> name, version, description of the chart
  values.yaml  -> default configuration (the public API of your chart)
  templates/   -> Kubernetes YAML with {{ .Values.xxx }} placeholders
```

---

## Learning Checklist

Work through these in order. Do not skip ahead.

### Kubernetes

- [ ] `kubectl get nodes` — cluster is reachable
- [ ] Create a Pod, describe it, read its logs
- [ ] Create a Deployment with 3 replicas, delete one pod, watch it restart
- [ ] Expose the Deployment with a Service, use `port-forward` to reach it locally
- [ ] Create a ConfigMap, inject its keys as environment variables into a Pod
- [ ] Create a namespace, deploy a workload into it

### Helm Basics

- [ ] `helm install my-nginx bitnami/nginx` — install from a repo
- [ ] `helm list`, `helm status`, `helm get values` — inspect a release
- [ ] `helm create my-app` — scaffold your own chart, read every generated file
- [ ] `helm install my-release ./my-app` — install your chart
- [ ] `helm upgrade my-release ./my-app --set replicaCount=3` — change a value
- [ ] `helm history my-release` — see revisions
- [ ] `helm rollback my-release 1` — roll back
- [ ] `helm template my-release ./my-app` — see the rendered output before deploying
- [ ] `helm lint ./my-app` — validate your chart
- [ ] `helm uninstall my-release`

### Helm Intermediate

- [ ] Create `values-dev.yaml` and `values-prod.yaml`, deploy with `-f`
- [ ] Use `helm upgrade --install` — understand why it is the standard CI/CD pattern
- [ ] Add `--atomic` to an upgrade, break the deploy deliberately, watch it roll back
- [ ] Write a `pre-upgrade` hook that runs a Job before the new pods start
- [ ] Add `bitnami/postgresql` as a chart dependency and deploy the whole stack
- [ ] `helm package ./my-app` — produce a versioned .tgz

---

## Useful Links

| Resource | URL |
|----------|-----|
| Helm docs | https://helm.sh/docs |
| Artifact Hub | https://artifacthub.io |
| Bitnami charts | https://github.com/bitnami/charts |
| Helm on GitHub | https://github.com/helm/helm |
| KIND docs | https://kind.sigs.k8s.io |
| Kubernetes docs | https://kubernetes.io/docs |
| Sprig functions | https://masterminds.github.io/sprig |
