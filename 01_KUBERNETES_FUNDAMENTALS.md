# Kubernetes — From First Principles

> If you understand why something exists, how it works becomes obvious.

---

## Table of Contents

1. [The Problem Kubernetes Solves](#1-the-problem-kubernetes-solves)
2. [Cluster Architecture](#2-cluster-architecture)
3. [Core Objects](#3-core-objects)
4. [Networking](#4-networking)
5. [Storage](#5-storage)
6. [Configuration Management](#6-configuration-management)
7. [The Full Request Flow](#7-the-full-request-flow)
8. [Namespaces and RBAC](#8-namespaces-and-rbac)
9. [kubectl Reference](#9-kubectl-reference)

---

## 1. The Problem Kubernetes Solves

### Before containers

You deploy your app on a server. It works. Then reality hits:
- Server crashes — app dies, no one restarts it
- Traffic spikes — single instance falls over
- New version ships — you take downtime to deploy
- You have 10 servers — you SSH into each one manually

### With Docker alone

Your app is packaged consistently. But:
- Who restarts the container if it crashes?
- Who runs 3 copies for load balancing?
- How do you deploy a new version without taking everything down?
- How do containers on different machines find each other?

### What Kubernetes adds

```
+-------------------------------------------------------+
|              Kubernetes — Container Orchestrator       |
|                                                       |
|  Self-healing      restarts crashed containers        |
|  Scaling           runs N replicas, adjusts to load   |
|  Rolling updates   deploys new versions with no downtime |
|  Load balancing    spreads traffic across all replicas |
|  Service discovery containers find each other by name |
+-------------------------------------------------------+
```

---

## 2. Cluster Architecture

A Kubernetes cluster has two kinds of machines: the control plane (the brain) and worker nodes (where your containers actually run).

```
+-----------------------------------------------------------+
|                    KUBERNETES CLUSTER                     |
|                                                           |
|  +-------------------------+                              |
|  |      CONTROL PLANE      |                              |
|  |                         |                              |
|  |  API Server             |  <- every request goes here  |
|  |  etcd                   |  <- stores all cluster state |
|  |  Scheduler              |  <- decides which node       |
|  |  Controller Manager     |  <- watches & reconciles     |
|  +-------------------------+                              |
|                                                           |
|  +-----------+  +-----------+  +-----------+             |
|  | WORKER 1  |  | WORKER 2  |  | WORKER 3  |             |
|  |           |  |           |  |           |             |
|  | kubelet   |  | kubelet   |  | kubelet   |             |
|  | kube-proxy|  | kube-proxy|  | kube-proxy|             |
|  | [Pod][Pod]|  | [Pod]     |  | [Pod][Pod]|             |
|  +-----------+  +-----------+  +-----------+             |
+-----------------------------------------------------------+
```

### What each component does

| Component | Responsibility |
|-----------|----------------|
| API Server | The single entry point — kubectl, Helm, everything talks here |
| etcd | Distributed key-value store, holds the desired and actual state |
| Scheduler | When a new Pod needs to run, picks the best worker node |
| Controller Manager | Constantly checks actual state vs desired state and reconciles |
| kubelet | Agent on each worker, receives instructions, starts/stops containers |
| kube-proxy | Manages iptables rules on each node for service networking |

### KIND specifically

```
Your Mac
   |
   v
Docker Desktop
   |
   v
KIND container  <- this single container acts as both control plane and worker
   |
   v
kubectl communicates via localhost on a randomly assigned port
```

KIND = Kubernetes IN Docker. It gives you a real cluster locally with no VMs.

---

## 3. Core Objects

### 3.1 Pod

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share a network stack and storage.

```
+--------------------------------------+
|                 POD                  |
|                                      |
|  +-------------+  +-------------+   |
|  | Container 1 |  | Container 2 |   |
|  | (your app)  |  | (sidecar)   |   |
|  +-------------+  +-------------+   |
|                                      |
|  - Both share the same IP address    |
|  - They talk to each other via localhost |
|  - They share mounted volumes        |
+--------------------------------------+
        IP: 10.244.0.5
```

> Pods are ephemeral. When a Pod dies and is replaced, it gets a completely new IP address. Never hardcode a Pod's IP anywhere.

```yaml
# pod.yaml
# You rarely write this directly — Deployment manages Pods for you.
# This is here so you understand what a Deployment creates underneath.

apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app   # labels are arbitrary key-value tags used for selection
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"   # 250m = 250 millicores = 0.25 of one CPU core
        limits:
          memory: "128Mi"
          cpu: "500m"
```

```bash
kubectl get pods
kubectl describe pod my-pod    # shows full spec + event log — very useful for debugging
kubectl logs my-pod            # stdout from the container
kubectl exec -it my-pod -- sh  # open a shell inside the container
kubectl delete pod my-pod      # if created by a Deployment, it will be recreated
```

---

### 3.2 Deployment

A Deployment says: "I want exactly N replicas of this Pod running at all times."

```
+------------------------------------------------------+
|                    DEPLOYMENT                        |
|  desired: replicas=3, image=nginx:1.25               |
|                                                      |
|         owns a ReplicaSet                            |
|              |                                       |
|     +---------+----------+                           |
|     v         v          v                           |
|   Pod-1     Pod-2      Pod-3    <- all identical     |
+------------------------------------------------------+

When Pod-2 crashes:
  Controller Manager: actual=2, desired=3
  Scheduler: assigns Pod-4 to an available node
  kubelet: pulls image, starts container
  Result: back to 3 replicas automatically
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3

  # The selector must match the labels on the Pod template below.
  # Kubernetes uses this to know which Pods belong to this Deployment.
  selector:
    matchLabels:
      app: my-app

  strategy:
    type: RollingUpdate   # replace pods gradually, not all at once
    rollingUpdate:
      maxSurge: 1         # allow 1 extra pod above desired count during update
      maxUnavailable: 0   # never reduce below desired count during update

  template:
    metadata:
      labels:
        app: my-app   # must match selector.matchLabels above
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl rollout status deployment/my-app    # watch the rollout live
kubectl rollout history deployment/my-app   # see past versions
kubectl rollout undo deployment/my-app      # rollback one step
kubectl scale deployment my-app --replicas=5
```

---

### Rolling Update — How Zero-Downtime Deploy Works

With `maxUnavailable: 0` and `maxSurge: 1`, here is what happens when you update to a new image:

```
Initial state (3 pods, old version):
  [v1]  [v1]  [v1]

Start 1 new pod (maxSurge allows going to 4):
  [v1]  [v1]  [v1]  [v2]

v2 passes readiness probe, terminate one v1:
  [v1]  [v1]  [v2]

Start another new pod:
  [v1]  [v1]  [v2]  [v2]

v2 ready, terminate another v1:
  [v1]  [v2]  [v2]

Continue until done:
  [v2]  [v2]  [v2]   <- zero downtime throughout
```

Traffic is only sent to pods that pass their readiness probe, so users never hit a pod that is not ready.

---

### 3.3 Service

Pods restart with new IPs. A Service gives a stable DNS name and virtual IP that always routes to the right pods, regardless of which specific pods are currently alive.

```
Client request
       |
       v
  +-----------------+
  |     SERVICE     |   stable DNS: my-app-svc.default.svc.cluster.local
  |  my-app-svc     |   stable IP:  10.96.45.100
  +---------+-------+
            |
            |  routes to pods matching label: app=my-app
            |
   +--------+--------+--------+
   v                 v        v
 Pod-1            Pod-2     Pod-3      <- round-robin load balanced
10.244.0.5     10.244.0.6  10.244.1.2
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  # Route to any pod that has this label.
  # When pods restart with new IPs, they still have this label.
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80        # port that clients connect to on the Service
      targetPort: 80  # port that the container is actually listening on
  type: ClusterIP     # only accessible inside the cluster
```

### Service Types

```
ClusterIP  (default)
  Reachable only within the cluster.
  Use this for microservices that talk to each other.

  [Frontend Pod] -> [backend-svc:80] -> [Backend Pods]


NodePort
  Opens a port (30000-32767) on every node.
  Access via <NodeIP>:<NodePort> from outside.
  Useful for local testing. Not for production.

  Browser -> Node IP:30080 -> Service -> Pods


LoadBalancer
  Provisions a cloud load balancer (AWS ALB, GCP LB, etc).
  Gives you an external IP. This is how production apps are exposed.

  Internet -> Cloud LB -> Service -> Pods


Ingress  (not a Service type, but handles external HTTP/HTTPS)
  One entry point, routes to many services by path or hostname.

  Internet
     |
  Ingress Controller (nginx, traefik, etc.)
     |-- /api   -> api-service  -> API Pods
     |-- /web   -> web-service  -> Web Pods
     |-- /auth  -> auth-service -> Auth Pods
```

```bash
kubectl get svc
kubectl describe svc my-app-svc
kubectl port-forward svc/my-app-svc 8080:80   # forward to your laptop for testing
```

---

### 3.4 ConfigMap

The goal: the same Docker image should work in dev, staging, and production. The only thing that changes is configuration. ConfigMap externalises that config.

```
Without ConfigMap:
  Image has DB_HOST=localhost baked in.
  You need a different image per environment.

With ConfigMap:
  Image reads DB_HOST from an environment variable.
  Each environment has its own ConfigMap.
  One image works everywhere.
```

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value pairs become environment variables
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  LOG_LEVEL: "info"

  # You can also store entire files as values
  config.json: |
    {
      "feature_x": true,
      "timeout_seconds": 30
    }
```

```yaml
# Use the ConfigMap in a Pod:
spec:
  containers:
    - name: app
      image: my-app:1.0
      # inject ALL keys from the ConfigMap as environment variables
      envFrom:
        - configMapRef:
            name: app-config
      # mount config.json as a file at /etc/config/config.json
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

---

### 3.5 Secret

Works exactly like ConfigMap, but for sensitive data. Values must be base64-encoded in YAML. Kubernetes stores them as Secrets and can restrict who can read them.

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  # echo -n "password123" | base64
  DB_PASSWORD: cGFzc3dvcmQxMjM=
  API_KEY: c2VjcmV0a2V5
```

```bash
# Easier than writing YAML: create from the command line
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD=password123 \
  --from-literal=API_KEY=secretkey

kubectl get secret app-secrets
kubectl describe secret app-secrets   # values are hidden in output
```

> Note: base64 is encoding, not encryption. Raw Kubernetes Secrets are not secure on their own. In production, use Sealed Secrets, HashiCorp Vault, or AWS Secrets Manager.

---

### 3.6 Namespace

Namespaces are virtual partitions inside a single cluster. Resources in different namespaces are isolated from each other by default.

```
+-----------------------------------------------+
|                   CLUSTER                     |
|                                               |
|  +-----------+     +-----------+              |
|  | default   |     |kube-system|              |
|  |           |     |           |              |
|  | your work |     | k8s internals             |
|  +-----------+     +-----------+              |
|                                               |
|  +-----------+     +-----------+              |
|  |    dev    |     |   prod    |              |
|  |           |     |           |              |
|  | dev builds|     | live app  |              |
|  +-----------+     +-----------+              |
+-----------------------------------------------+
```

```bash
kubectl create namespace dev
kubectl apply -f deployment.yaml -n dev
kubectl get all -n dev
kubectl get pods -n dev
```

---

### 3.7 Other Objects Worth Knowing

#### HorizontalPodAutoscaler (HPA)

Watches CPU and memory, scales replicas up and down automatically.

```
Low traffic  ->  HPA scales to 2 replicas
High traffic ->  HPA scales to 8 replicas
```

```bash
# Scale between 2 and 10 pods, targeting 70% average CPU
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70
```

#### PersistentVolume (PV) and PersistentVolumeClaim (PVC)

Pods are stateless by default — data written inside a container is lost when it restarts. PVs provide durable storage.

```
[Pod]
  |
  v
[PersistentVolumeClaim]   <- the Pod requests: "I need 10Gi"
  |
  v
[PersistentVolume]        <- the actual provisioned disk
  |
  v
[Cloud disk / NFS / etc]  <- physical storage
```

#### Jobs and CronJobs

- **Job**: runs a container to completion. Used for DB migrations, data imports.
- **CronJob**: runs a Job on a schedule. Used for backups, reports, cleanups.

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
spec:
  schedule: "0 0 * * *"   # every day at midnight (cron syntax)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: my-cleanup-job:1.0
          restartPolicy: OnFailure
```

---

## 4. Networking

### Pod-to-Pod

Every Pod gets a unique IP. Any Pod can reach any other Pod directly, across nodes, with no NAT. This is handled by the CNI plugin (Flannel, Calico, Cilium, etc.).

```
Pod A (10.244.0.5) on Node 1  ---->  Pod B (10.244.1.3) on Node 2

This works without any extra configuration.
```

### DNS

Every Service gets an automatic DNS entry:

```
Service: my-backend   in namespace: default

Full DNS name:   my-backend.default.svc.cluster.local
Short name:      my-backend   (works inside the same namespace)

Your app code just calls:  http://my-backend/api
Kubernetes routes it to the correct pods automatically.
```

---

## 5. Storage

```
emptyDir         -> temporary, tied to pod lifetime, gone on restart
hostPath         -> mounts a directory from the node's filesystem (risky)
PersistentVolume -> backed by real storage, survives pod restarts

Typical Postgres setup:

[Postgres Pod]
      |
      v
[PersistentVolumeClaim]   <- "I need 20Gi, ReadWriteOnce"
      |
      v
[PersistentVolume]        <- provisioned disk
      |
      v
[AWS EBS / GCP PD / etc]
```

---

## 6. Configuration Management

The principle: one Docker image, multiple environments. Configuration is injected at runtime, not baked in.

```
dev:
  ConfigMap: DB_HOST=dev-postgres-svc, LOG_LEVEL=debug
  Secret:    DB_PASS=devpass

staging:
  ConfigMap: DB_HOST=staging-postgres-svc, LOG_LEVEL=info
  Secret:    DB_PASS=stagingpass

prod:
  ConfigMap: DB_HOST=prod-postgres-svc, LOG_LEVEL=error
  Secret:    DB_PASS=<fetched from Vault at deploy time>

Same image deployed everywhere. Zero image rebuilds for config changes.
```

---

## 7. The Full Request Flow

Tracing a browser request all the way through to Postgres:

```
Browser visits https://myapp.example.com
      |
      v
DNS resolves to Load Balancer IP
      |
      v
Cloud Load Balancer (or NodePort in KIND)
      |
      v
Ingress Controller (nginx)
  - terminates TLS
  - matches /api  ->  api-service
      |
      v
Service: api-service
  - stable IP, uses label selector to find pods
      |
      v
One of the API Pods  (round-robin)
  - your application code runs here
  - needs data, calls postgres-svc by DNS name
      |
      v
Service: postgres-svc
      |
      v
Postgres Pod
```

---

## 8. Namespaces and RBAC

### RBAC — Role-Based Access Control

Controls who can do what inside the cluster.

```
Role            -> a list of allowed actions in one namespace
ClusterRole     -> same, but applies across all namespaces
RoleBinding     -> binds a Role to a user or ServiceAccount
ClusterRoleBinding -> binds a ClusterRole to a user or ServiceAccount
```

```yaml
# Give a "developer" user read-only access to pods in the "dev" namespace.
# Two objects needed: the Role (what is allowed) and the RoleBinding (who gets it).

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]              # "" = the core API group (pods, services, etc.)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]   # read-only — no create, delete, patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-reader
  namespace: dev
subjects:
  - kind: User
    name: developer         # must match the username in the kubeconfig cert
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 9. kubectl Reference

```bash
# -- CLUSTER ----------------------------------------------------------------
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide          # includes IP, OS, kernel version

# -- PODS -------------------------------------------------------------------
kubectl get pods                   # current namespace
kubectl get pods -A                # all namespaces
kubectl get pods -w                # watch for changes (live)
kubectl get pods -o wide           # include node name and pod IP

kubectl describe pod <name>        # full spec + event log, use when debugging
kubectl logs <pod>                 # stdout of the container
kubectl logs <pod> -f              # follow log output (like tail -f)
kubectl logs <pod> -c <container>  # if the pod has multiple containers
kubectl logs <pod> --previous      # logs from a crashed previous instance

kubectl exec -it <pod> -- sh       # shell into the container
kubectl exec -it <pod> -- bash     # if the image has bash

kubectl delete pod <name>          # pod managed by a Deployment will restart

# -- DEPLOYMENTS ------------------------------------------------------------
kubectl get deployments
kubectl describe deployment <name>
kubectl rollout status deployment/<name>   # watch pods come up live
kubectl rollout history deployment/<name>  # list previous revisions
kubectl rollout undo deployment/<name>     # rollback one revision
kubectl rollout undo deployment/<name> --to-revision=3

kubectl scale deployment <name> --replicas=5
kubectl set image deployment/<name> <container>=nginx:1.26   # update image

# -- SERVICES ---------------------------------------------------------------
kubectl get svc
kubectl describe svc <name>
kubectl port-forward svc/<name> 8080:80   # expose locally on port 8080

# -- CONFIGMAPS AND SECRETS -------------------------------------------------
kubectl get configmap
kubectl describe configmap <name>
kubectl get secret
kubectl get secret <name> -o yaml        # shows base64-encoded values

# -- APPLY AND DELETE -------------------------------------------------------
kubectl apply -f file.yaml               # create or update
kubectl apply -f ./manifests/            # apply every .yaml in a directory
kubectl delete -f file.yaml              # delete what was created by this file

# -- NAMESPACES -------------------------------------------------------------
kubectl get ns
kubectl create namespace staging
kubectl get pods -n staging
kubectl config set-context --current --namespace=staging   # change default ns

# -- DEBUGGING --------------------------------------------------------------
kubectl get events --sort-by=.metadata.creationTimestamp   # recent cluster events
kubectl top pods          # CPU and memory (requires metrics-server)
kubectl top nodes
kubectl api-resources     # list all resource types available in the cluster
```

---

> Next: [02_HELM_COMPLETE_GUIDE.md](./02_HELM_COMPLETE_GUIDE.md)
