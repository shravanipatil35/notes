# Kubernetes — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Kubernetes vs Docker / Docker Swarm](#2-kubernetes-vs-docker--docker-swarm)
3. [Kubernetes Architecture](#3-kubernetes-architecture)
4. [Core Concepts / Objects](#4-core-concepts--objects)
5. [kubectl Basics](#5-kubectl-basics)
6. [YAML Manifests — Structure](#6-yaml-manifests--structure)
7. [Pods Deep Dive](#7-pods-deep-dive)
8. [Deployments & ReplicaSets](#8-deployments--replicasets)
9. [Services & Networking](#9-services--networking)
10. [ConfigMaps & Secrets](#10-configmaps--secrets)
11. [Volumes & Persistent Storage](#11-volumes--persistent-storage)
12. [Namespaces](#12-namespaces)
13. [Health Checks (Probes)](#13-health-checks-probes)
14. [Resource Requests & Limits](#14-resource-requests--limits)
15. [Horizontal Pod Autoscaler (HPA)](#15-horizontal-pod-autoscaler-hpa)
16. [StatefulSets](#16-statefulsets)
17. [DaemonSets](#17-daemonsets)
18. [Jobs & CronJobs](#18-jobs--cronjobs)
19. [RBAC & Security](#19-rbac--security)
20. [Helm](#20-helm)
21. [Advanced Scheduling](#21-advanced-scheduling)
22. [Troubleshooting](#22-troubleshooting)
23. [Cheat Sheet](#23-cheat-sheet)
24. [Interview Questions & Answers](#24-interview-questions--answers)

---

## 1. Introduction

**Kubernetes (K8s)** is an open-source **container orchestration platform** originally developed by Google (based on their internal system "Borg"), now maintained by the CNCF. It automates deployment, scaling, networking, and management of containerized applications across a cluster of machines.

**Why Kubernetes?**
- Automates scaling, healing, and rollouts/rollbacks
- Manages applications across many hosts (not just one machine like plain Docker)
- Declarative — you describe the desired state, K8s continuously works to match reality to it
- Self-healing — restarts failed containers, reschedules pods from dead nodes
- Built-in service discovery and load balancing
- Industry-standard for production container orchestration

---

## 2. Kubernetes vs Docker / Docker Swarm

| Aspect | Docker (alone) | Docker Swarm | Kubernetes |
|---|---|---|---|
| Scope | Single host | Multi-host, simple | Multi-host, advanced |
| Learning curve | Easy | Easy | Steep |
| Self-healing | No | Basic | Advanced |
| Auto-scaling | No | Manual | Built-in (HPA/VPA/Cluster Autoscaler) |
| Networking | Simple | Built-in overlay | Highly configurable (CNI plugins) |
| Config | docker run / Compose | Stack files | YAML manifests |
| Ecosystem | Small | Small | Huge (Helm, Operators, Service Mesh) |

**Important nuance:** Kubernetes doesn't run "Docker containers" directly anymore — it uses the **Container Runtime Interface (CRI)** to talk to any OCI-compliant runtime (commonly **containerd** or **CRI-O**). Docker Engine itself was removed as a direct runtime (`dockershim` deprecated in 2022), but images built by Docker still run fine since they follow the OCI image spec.

---

## 3. Kubernetes Architecture

A cluster = **Control Plane** (brain) + **Worker Nodes** (where apps actually run).

```
                         ┌─────────────────────────────────────────┐
                         │              CONTROL PLANE                │
                         │                                            │
                         │  ┌──────────────┐   ┌───────────────────┐ │
                         │  │ API Server    │   │  etcd (key-value   │ │
                         │  │ (kube-apiserver) │ store - cluster     │ │
                         │  └──────┬───────┘   │  state)             │ │
                         │         │            └───────────────────┘ │
                         │  ┌──────▼───────┐   ┌───────────────────┐ │
                         │  │ Scheduler     │   │ Controller         │ │
                         │  │ (kube-        │   │ Manager             │ │
                         │  │  scheduler)   │   │ (kube-controller-   │ │
                         │  └───────────────┘   │  manager)           │ │
                         │                       └───────────────────┘ │
                         └─────────────────────────────────────────┘
                                          │
                  ┌───────────────────────┼───────────────────────┐
                  ▼                       ▼                       ▼
          ┌───────────────┐      ┌───────────────┐       ┌───────────────┐
          │  Worker Node 1  │      │  Worker Node 2  │       │  Worker Node 3  │
          │ ┌────────────┐ │      │ ┌────────────┐ │       │ ┌────────────┐ │
          │ │  kubelet    │ │      │ │  kubelet    │ │       │ │  kubelet    │ │
          │ │  kube-proxy │ │      │ │  kube-proxy │ │       │ │  kube-proxy │ │
          │ │  Pods...    │ │      │ │  Pods...    │ │       │ │  Pods...    │ │
          │ │  container  │ │      │ │  container  │ │       │ │  container  │ │
          │ │  runtime    │ │      │ │  runtime    │ │       │ │  runtime    │ │
          │ └────────────┘ │      │ └────────────┘ │       │ └────────────┘ │
          └───────────────┘      └───────────────┘       └───────────────┘
```

### Control Plane Components
- **kube-apiserver** — front door to the cluster; all communication (kubectl, controllers, kubelets) goes through its REST API.
- **etcd** — distributed, consistent key-value store holding all cluster state/config (the "source of truth").
- **kube-scheduler** — watches for newly created Pods with no node assigned, picks the best node based on resource requirements, constraints, affinity rules.
- **kube-controller-manager** — runs controller loops (Node controller, ReplicaSet controller, Deployment controller, etc.) that continuously reconcile actual state → desired state.
- **cloud-controller-manager** — integrates with cloud provider APIs (load balancers, storage, node lifecycle) — only present in cloud-managed clusters.

### Worker Node Components
- **kubelet** — agent on each node; talks to API server, ensures containers described in PodSpecs are running and healthy.
- **kube-proxy** — maintains network rules on nodes, enabling Service-based communication/load balancing (via iptables or IPVS).
- **Container runtime** — containerd, CRI-O, etc. — actually runs the containers.

---

## 4. Core Concepts / Objects

| Object | Purpose |
|---|---|
| **Pod** | Smallest deployable unit; one or more tightly-coupled containers sharing network/storage |
| **ReplicaSet** | Ensures a specified number of identical Pod replicas are running |
| **Deployment** | Manages ReplicaSets; provides declarative updates, rollbacks, rolling updates |
| **Service** | Stable network endpoint (virtual IP + DNS name) for accessing a set of Pods |
| **Ingress** | Manages external HTTP/HTTPS access to Services, with routing rules |
| **ConfigMap** | Stores non-sensitive configuration data as key-value pairs |
| **Secret** | Stores sensitive data (passwords, tokens, keys), base64-encoded |
| **Namespace** | Virtual cluster for logically isolating resources |
| **Volume / PersistentVolume (PV)** | Storage abstraction for Pods |
| **PersistentVolumeClaim (PVC)** | A request for storage by a user/Pod |
| **StatefulSet** | Manages stateful apps needing stable identity/storage (e.g., databases) |
| **DaemonSet** | Ensures a copy of a Pod runs on every (or selected) node |
| **Job / CronJob** | Run-to-completion / scheduled tasks |
| **Node** | A worker machine (VM or physical) in the cluster |
| **Label/Selector** | Key-value tags used to organize and select groups of objects |

---

## 5. kubectl Basics

```bash
kubectl version
kubectl cluster-info
kubectl get nodes
kubectl get pods
kubectl get pods -A                      # all namespaces
kubectl get all                          # pods, services, deployments, etc.
kubectl describe pod <pod-name>          # detailed info + events
kubectl logs <pod-name>
kubectl logs -f <pod-name>               # follow
kubectl logs <pod-name> -c <container>   # multi-container pod
kubectl exec -it <pod-name> -- bash
kubectl apply -f manifest.yaml           # create/update from file (declarative)
kubectl delete -f manifest.yaml
kubectl create -f manifest.yaml          # create (imperative-ish)
kubectl delete pod <pod-name>
kubectl edit deployment <name>           # live edit in editor
kubectl scale deployment <name> --replicas=5
kubectl rollout status deployment <name>
kubectl rollout undo deployment <name>
kubectl rollout history deployment <name>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top nodes                        # requires metrics-server
kubectl top pods
kubectl port-forward pod/<name> 8080:80
kubectl config get-contexts
kubectl config use-context <name>
kubectl explain pod.spec.containers      # inline API docs
```

---

## 6. YAML Manifests — Structure

Every K8s object manifest has 4 top-level fields:
```yaml
apiVersion: apps/v1     # which API version
kind: Deployment        # what kind of object
metadata:               # name, labels, namespace
  name: my-app
  labels:
    app: my-app
spec:                   # desired state (object-specific)
  ...
status:                 # current state (filled in by K8s, not by you)
```

---

## 7. Pods Deep Dive

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
  - name: app-container
    image: myapp:1.0
    ports:
    - containerPort: 8080
    env:
    - name: ENV_MODE
      value: "production"
    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 15
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

**Key facts:**
- Pods are **ephemeral** — they get a new IP every time they're recreated. You almost never manage Pods directly in production; use a Deployment/StatefulSet/DaemonSet to manage them.
- A Pod can have **multiple containers** sharing the same network namespace (localhost) and volumes — common pattern: **sidecar containers** (e.g., logging agent, service mesh proxy like Envoy).
- **Init containers** run to completion *before* the main containers start (e.g., wait for a dependency, run migrations).

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
  - name: app
    image: myapp:1.0
```

---

## 8. Deployments & ReplicaSets

A **Deployment** is the standard way to run stateless apps — it manages ReplicaSets, which manage Pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

**Update strategies:**
- **RollingUpdate** (default) — gradually replaces old Pods with new ones, zero downtime
- **Recreate** — kills all old Pods, then creates new ones (downtime, but avoids old/new version overlap)

```bash
kubectl set image deployment/my-app my-app=myapp:2.0   # trigger rolling update
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app                  # rollback to previous version
kubectl rollout undo deployment/my-app --to-revision=2
```

**Why Deployment → ReplicaSet → Pod (3 layers)?** The Deployment manages ReplicaSets to enable rolling updates and rollbacks — each new version gets a new ReplicaSet, and old ones are kept around (scaled to 0) so you can roll back instantly.

---

## 9. Services & Networking

A **Service** gives a stable virtual IP/DNS name to a dynamic set of Pods (selected via labels), since Pod IPs change constantly.

| Type | Description |
|---|---|
| **ClusterIP** (default) | Internal-only virtual IP, reachable within the cluster |
| **NodePort** | Exposes the service on a static port on every node's IP (30000-32767 range) |
| **LoadBalancer** | Provisions an external cloud load balancer (cloud-provider dependent) |
| **ExternalName** | Maps the service to an external DNS name (CNAME) |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP
  selector:
    app: my-app          # routes traffic to Pods matching this label
  ports:
  - protocol: TCP
    port: 80              # service port
    targetPort: 8080       # container port
```

**DNS:** Every Service automatically gets a DNS entry: `my-app-service.my-namespace.svc.cluster.local` — Pods can reach it just by hostname `my-app-service` within the same namespace.

**Ingress** — manages external HTTP(S) routing to multiple Services using host/path rules, usually backed by an Ingress Controller (NGINX, Traefik, AWS ALB Ingress, etc.):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
```

---

## 10. ConfigMaps & Secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=   # base64 encoded (NOT encrypted by default!)
```

```bash
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=supersecret
kubectl create configmap app-config --from-file=config.properties
```

Using them in a Pod:
```yaml
containers:
- name: app
  image: myapp:1.0
  envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secret
  # or individually:
  env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
  volumeMounts:
  - name: secret-vol
    mountPath: /etc/secrets
volumes:
- name: secret-vol
  secret:
    secretName: app-secret
```

**Important interview point:** Secrets are base64-**encoded**, not encrypted, by default — base64 is trivially reversible. For real protection you need **encryption at rest** (enabled at the etcd/API server level) and/or an external secret manager (Vault, AWS Secrets Manager, Sealed Secrets, External Secrets Operator).

---

## 11. Volumes & Persistent Storage

| Concept | Description |
|---|---|
| **Volume** | Storage attached to a Pod's lifecycle (many types: emptyDir, hostPath, configMap, secret, etc.) |
| **PersistentVolume (PV)** | A piece of storage provisioned in the cluster (by admin or dynamically), independent of any Pod |
| **PersistentVolumeClaim (PVC)** | A request for storage by a user — binds to a matching PV |
| **StorageClass** | Defines a "class" of storage for dynamic provisioning (e.g., `gp3`, `fast-ssd`) |

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: gp3
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        volumeMounts:
        - mountPath: "/data"
          name: storage
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: my-pvc
```

**Access modes:**
- `ReadWriteOnce` (RWO) — single node read/write
- `ReadOnlyMany` (ROX) — many nodes read-only
- `ReadWriteMany` (RWX) — many nodes read/write (needs NFS/cloud-file-storage-backed PV)

---

## 12. Namespaces

Logical partitioning of a cluster — used for multi-team/multi-environment isolation (not strong security isolation by itself).

```bash
kubectl create namespace dev
kubectl get pods -n dev
kubectl config set-context --current --namespace=dev
```

```yaml
metadata:
  name: my-app
  namespace: dev
```

Default namespaces: `default`, `kube-system` (system components), `kube-public`, `kube-node-lease`.

---

## 13. Health Checks (Probes)

| Probe | Purpose |
|---|---|
| **livenessProbe** | "Is the container alive?" — if it fails, kubelet **restarts** the container |
| **readinessProbe** | "Is the container ready to receive traffic?" — if it fails, Pod is removed from Service endpoints (but not restarted) |
| **startupProbe** | "Has the app finished starting?" — disables liveness/readiness checks until it succeeds; useful for slow-starting apps |

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

Probe types: `httpGet`, `tcpSocket`, `exec` (run a command, success = exit code 0).

---

## 14. Resource Requests & Limits

```yaml
resources:
  requests:        # guaranteed minimum — used by scheduler for placement decisions
    cpu: "250m"     # 250 millicores = 0.25 CPU
    memory: "128Mi"
  limits:          # hard ceiling — container is throttled (CPU) or OOM-killed (memory) if exceeded
    cpu: "500m"
    memory: "256Mi"
```

**QoS Classes (derived automatically from requests/limits):**
- **Guaranteed** — requests == limits for all containers → highest priority, last to be evicted
- **Burstable** — requests < limits → medium priority
- **BestEffort** — no requests/limits set → lowest priority, first to be evicted under pressure

---

## 15. Horizontal Pod Autoscaler (HPA)

Automatically scales the number of Pod replicas based on observed metrics (CPU, memory, or custom metrics via Prometheus adapter).

```bash
kubectl autoscale deployment my-app --cpu-percent=70 --min=2 --max=10
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Related: **VPA** (Vertical Pod Autoscaler — adjusts requests/limits) and **Cluster Autoscaler** (adds/removes nodes based on pending/unschedulable Pods).

---

## 16. StatefulSets

For applications needing **stable network identity** and **stable persistent storage** per replica (databases, Kafka, Zookeeper, Elasticsearch).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**Key differences vs Deployment:**
- Pods get **stable, predictable names**: `mysql-0`, `mysql-1`, `mysql-2` (not random hashes)
- Each Pod gets its **own PVC** (via `volumeClaimTemplates`) that persists even if the Pod is rescheduled
- Pods are created/deleted/updated in **strict ordinal order** (0, 1, 2...), not in parallel

---

## 17. DaemonSets

Ensures one Pod copy runs on **every node** (or a selected subset) — used for node-level agents: log collectors (Fluentd), monitoring agents (Node Exporter), CNI plugins, storage daemons.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: fluentd
        image: fluentd:latest
```

---

## 18. Jobs & CronJobs

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 1
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:1.0
        command: ["python", "migrate.py"]
      restartPolicy: Never
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"      # standard cron syntax — 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:1.0
          restartPolicy: OnFailure
```

---

## 19. RBAC & Security

**RBAC (Role-Based Access Control)** controls who can do what in the cluster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                       # namespace-scoped (use ClusterRole for cluster-wide)
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Other security mechanisms:**
- **ServiceAccounts** — identity for processes running inside Pods (to talk to the API server)
- **Network Policies** — firewall rules controlling Pod-to-Pod traffic (requires a CNI plugin that supports it, e.g., Calico)
- **Pod Security Standards/Admission** — restricts privileged containers, host access, etc. (replaced the deprecated PodSecurityPolicy)
- **Secrets encryption at rest** — encrypts Secret data in etcd

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-except-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

---

## 20. Helm

**Helm** is the package manager for Kubernetes — bundles manifests into reusable, parameterized **charts**.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-release bitnami/postgresql
helm upgrade my-release bitnami/postgresql --set auth.password=secret
helm rollback my-release 1
helm uninstall my-release
helm list
helm create mychart        # scaffold a new chart
helm template mychart      # render templates locally without installing
```

**Chart structure:**
```
mychart/
  Chart.yaml          # metadata
  values.yaml          # default config values
  templates/            # YAML manifests with Go templating
    deployment.yaml
    service.yaml
  charts/                # dependency charts
```

---

## 21. Advanced Scheduling

**Node affinity / anti-affinity** — control which nodes Pods can/should run on:
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
```

**Pod affinity/anti-affinity** — schedule Pods near or away from other Pods:
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-app
      topologyKey: "kubernetes.io/hostname"   # spread replicas across different nodes
```

**Taints & Tolerations** — repel Pods from nodes unless they explicitly tolerate the taint:
```bash
kubectl taint nodes node1 key=value:NoSchedule
```
```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

**Custom Resource Definitions (CRDs) & Operators** — extend the K8s API with your own object types, and write controllers ("Operators") that manage complex stateful applications (e.g., the Postgres Operator, Prometheus Operator) using the same reconcile-loop pattern K8s itself uses.

---

## 22. Troubleshooting

```bash
kubectl get pods                          # check status (Pending/CrashLoopBackOff/etc.)
kubectl describe pod <pod>                # check Events section at the bottom — usually shows the real reason
kubectl logs <pod> --previous              # logs from the previous crashed instance
kubectl get events --sort-by=.lastTimestamp
kubectl exec -it <pod> -- sh
```

**Common Pod statuses & meaning:**
| Status | Likely cause |
|---|---|
| `Pending` | Can't be scheduled — insufficient resources, no matching node, unbound PVC |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, registry auth issue, network issue |
| `CrashLoopBackOff` | Container keeps crashing — check `kubectl logs --previous` |
| `OOMKilled` | Container exceeded memory limit |
| `ContainerCreating` (stuck) | Volume mount issue, network plugin issue |
| `Evicted` | Node ran low on resources, kubelet evicted the Pod |

---

## 23. Cheat Sheet

```bash
# Cluster
kubectl cluster-info
kubectl get nodes -o wide

# Resources
kubectl get pods,svc,deploy -A
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl get <resource> <name> -o yaml

# Debugging
kubectl describe pod <name>
kubectl logs <name> [-f] [--previous]
kubectl exec -it <name> -- bash

# Scaling/updates
kubectl scale deployment <name> --replicas=N
kubectl set image deployment/<name> <container>=<image>
kubectl rollout status/undo/history deployment/<name>

# Namespaces & context
kubectl config get-contexts / use-context <name>
kubectl get pods -n <namespace>

# Helm
helm install/upgrade/rollback/uninstall <release> <chart>
```

---

## 24. Interview Questions & Answers

**Q1: What is Kubernetes and what problem does it solve?**
A: An open-source container orchestration platform that automates deployment, scaling, networking, and self-healing of containerized applications across a cluster of machines — solving the problem of manually managing containers across many hosts.

**Q2: Explain the Kubernetes architecture (control plane vs worker nodes).**
A: The **control plane** (API server, etcd, scheduler, controller manager) makes global decisions about the cluster and stores its state. **Worker nodes** run the actual application Pods, managed locally by the **kubelet** (talks to API server, runs containers via the runtime) and **kube-proxy** (handles Service networking).

**Q3: What is a Pod, and why not just run containers directly?**
A: A Pod is the smallest deployable unit in K8s — one or more containers that share network namespace (same IP, can talk via localhost) and storage volumes. K8s schedules and manages Pods, not individual containers, because tightly-coupled containers (e.g., an app + a sidecar) often need to be co-located and treated as a single unit.

**Q4: Difference between a Deployment and a StatefulSet?**
A: Deployment manages stateless, interchangeable Pods with random names — any replica can be replaced freely. StatefulSet manages stateful apps needing stable network identity (predictable Pod names like `db-0`, `db-1`) and stable per-Pod persistent storage that survives rescheduling — Pods are created/updated in strict order.

**Q5: How does a Service work, and why is it needed?**
A: Pods are ephemeral and get new IPs whenever recreated, so you can't rely on direct Pod IPs. A Service provides a stable virtual IP and DNS name, and uses label selectors to dynamically route traffic to whichever Pods currently match — kube-proxy implements this via iptables/IPVS rules on each node.

**Q6: What's the difference between a livenessProbe and a readinessProbe?**
A: livenessProbe checks if the container is still functioning — failure causes a **restart**. readinessProbe checks if the container is ready to serve traffic — failure just **removes it from Service endpoints** temporarily (no restart), so it stops receiving traffic until it passes again.

**Q7: How does a rolling update work?**
A: The Deployment controller creates a new ReplicaSet for the new Pod template and gradually scales it up while scaling the old ReplicaSet down, governed by `maxSurge` (how many extra Pods can be created above desired count) and `maxUnavailable` (how many can be unavailable during the rollout) — achieving zero-downtime deployments.

**Q8: What's the difference between a ConfigMap and a Secret?**
A: Both store key-value configuration data injected into Pods as env vars or mounted files. ConfigMaps are for non-sensitive data; Secrets are intended for sensitive data, but by default are only **base64-encoded** (not encrypted) — real protection requires enabling encryption at rest and/or RBAC restrictions, or using an external secret manager.

**Q9: Explain requests vs limits for resources.**
A: `requests` is what the scheduler guarantees and uses to decide which node has room for the Pod. `limits` is the hard ceiling the container cannot exceed — CPU usage above the limit gets throttled, memory usage above the limit triggers an OOM kill. This combination also determines the Pod's QoS class (Guaranteed/Burstable/BestEffort), which affects eviction priority under resource pressure.

**Q10: What is a Horizontal Pod Autoscaler?**
A: A controller that automatically adjusts the number of Pod replicas in a Deployment/StatefulSet based on observed metrics (commonly CPU/memory utilization, or custom/external metrics), scaling out under load and back in when idle.

**Q11: What happens when a node fails?**
A: The node controller (via missed heartbeats) marks the node as `NotReady` after a timeout. After a grace period, Pods on that node are marked for eviction, and if they're managed by a Deployment/ReplicaSet/StatefulSet, the relevant controller schedules replacement Pods on healthy nodes — this is K8s's self-healing in action.

**Q12: What's the difference between ClusterIP, NodePort, and LoadBalancer Services?**
A: ClusterIP exposes the Service only internally within the cluster (default). NodePort additionally opens a static port (30000-32767) on every node's IP, making it reachable from outside the cluster. LoadBalancer provisions an external cloud load balancer pointing at the Service (cloud-provider-dependent), typically used for production external exposure — often paired with/replaced by an Ingress for HTTP routing.

**Q13: What is an Ingress and how is it different from a Service?**
A: A Service provides L4 (TCP/UDP) load balancing/exposure. An Ingress operates at L7 (HTTP/HTTPS), routing external traffic to different Services based on hostnames/paths, and can handle TLS termination — all through a single external entry point managed by an Ingress Controller (e.g., NGINX Ingress).

**Q14: What's the difference between a DaemonSet and a Deployment?**
A: A Deployment runs a specified *number* of replicas, scheduled wherever the scheduler decides is best. A DaemonSet ensures exactly *one* Pod runs on every (or a selected subset of) node — used for node-level agents like log collectors or monitoring daemons, not general application scaling.

**Q15: How do you handle secrets/config across environments (dev/staging/prod)?**
A: Common approaches: separate Namespaces or clusters per environment with environment-specific ConfigMaps/Secrets, Helm charts with environment-specific `values.yaml` files, Kustomize overlays, or GitOps tools (ArgoCD/Flux) that sync environment-specific manifests from Git.

**Q16: What is etcd and why is it critical?**
A: etcd is a distributed, consistent key-value store that holds **all** cluster state — every object's desired and current state. It's the single source of truth; if etcd is lost or corrupted without backups, the entire cluster's configuration is lost. Production clusters run etcd in a highly-available, regularly-backed-up cluster (typically 3 or 5 nodes for quorum).

**Q17: What is the difference between a Role and a ClusterRole in RBAC?**
A: A `Role` grants permissions within a single namespace. A `ClusterRole` grants permissions cluster-wide (or can be bound within a specific namespace too) — used for things like cluster-level resources (Nodes, PersistentVolumes) or when the same role needs to apply across many namespaces.

**Q18: What is a CrashLoopBackOff and how do you debug it?**
A: It means a container keeps crashing and Kubernetes keeps restarting it with an increasing backoff delay. Debug by checking `kubectl logs <pod> --previous` (logs from the crashed instance), `kubectl describe pod` for events, verifying the command/entrypoint, checking for missing config/env vars, and checking resource limits (could be getting OOMKilled).

**Q19: What's the difference between a PersistentVolume and a PersistentVolumeClaim?**
A: A PersistentVolume (PV) is the actual piece of storage in the cluster (provisioned by an admin or dynamically via a StorageClass). A PersistentVolumeClaim (PVC) is a *request* for storage made by a user/Pod, specifying size and access mode — Kubernetes binds the PVC to a matching available PV.

**Q20: How would you achieve zero-downtime deployment in Kubernetes?**
A: Use a Deployment with `RollingUpdate` strategy, set appropriate `maxSurge`/`maxUnavailable`, configure a proper `readinessProbe` so traffic only reaches Pods once truly ready, and use `preStop` hooks / graceful termination periods (`terminationGracePeriodSeconds`) so in-flight requests complete before a Pod is killed during the rollout.

---

### Final interview tip
Be ready to **draw the control plane vs worker node architecture** from memory, explain the **Pod → ReplicaSet → Deployment** relationship, and walk through what happens **end-to-end when you run `kubectl apply` on a Deployment** (API server validates & persists to etcd → scheduler assigns Pods to nodes → kubelet on each node pulls the image and starts containers → kube-proxy updates routing → readiness probes determine when Service traffic flows). This end-to-end flow question comes up constantly.

---
*End of notes.*
