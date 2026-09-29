# Cluster API + ArgoCD GitOps Cluster Provisioning

Declarative, GitOps-driven provisioning of Kubernetes workload clusters (`c1`, `c2`, ...) using
[Cluster API](https://cluster-api.sigs.k8s.io/) (CAPI) with the Docker infrastructure provider (CAPD),
templated with Helm, and deployed via ArgoCD's app-of-apps pattern.

> **Environment note:** this setup uses CAPD (Docker-in-Docker nodes) for local development/learning.
> It is **not** meant for production. See [Known Limitations](#known-limitations-of-the-docker-provider) below.

---

## Architecture

```
                          ┌─────────────────────┐
                          │   Management Cluster │  (kind, "mgmt")
                          │   - ArgoCD            │
                          │   - Cluster API core  │
                          │   - CAPD provider     │
                          └──────────┬───────────┘
                                     │ reconciles
                    ┌────────────────┼────────────────┐
                    ▼                                 ▼
            ┌───────────────┐                 ┌───────────────┐
            │   Cluster c1   │                 │   Cluster c2   │
            │  (Docker nodes)│                 │  (Docker nodes)│
            └───────────────┘                 └───────────────┘
```

- **Management cluster (`mgmt`)**: a plain `kind` cluster hosting ArgoCD and all CAPI controllers
  (`capi-system`, `capi-kubeadm-bootstrap-system`, `capi-kubeadm-control-plane-system`, `capd-system`).
- **Workload clusters (`c1`, `c2`, ...)**: created *by* CAPI, running as CAPD-managed Docker containers,
  each with its own control plane, workers, and HAProxy load balancer (`<cluster>-lb`).
- **GitOps**: ArgoCD watches this repo. Every cluster's desired state (node counts, k8s version) lives
  in a per-cluster `values-*.yaml` file — nothing is created by hand.

---

## Repo Structure

```
.

├── cluster-api/                              # Helm chart — one Cluster per values file
│   ├── Chart.yaml
│   ├── values-c1.yaml
│   ├── values-c2.yaml
│   └── templates/
│       └── clusterapi-template.yaml               # Cluster, DockerCluster, KubeadmControlPlane,
│                                       # DockerMachineTemplate(s), KubeadmConfigTemplate,
│                                       # MachineDeployment, MachineHealthCheck (x2)
└── argocd-apps/                       # App-of-apps
    ├── root-app.yaml                  # the ONLY manifest applied by hand
    ├── c1-cluster-create.yaml
    └── c2-cluster-create.yaml
```

### Why `cluster-class/` is separate from the Helm chart

CAPI's `ClusterClass` patches use their own Go-template syntax (`{{ .builtin.controlPlane.version }}`),
evaluated by the CAPI controller at runtime — **not** by Helm. If these files live inside a Helm
`templates/` directory, Helm tries to render them at `helm template` time and fails. This project
avoids `ClusterClass` entirely in favor of flat, per-cluster manifests templated only with
`{{ .Values.cluster.* }}`, which sidesteps the conflict. `cluster-class/` is kept as a plain manifest
directory (no `Chart.yaml`) so ArgoCD applies it as-is, with no templating at all.

---

## Per-cluster configuration (`values-<name>.yaml`)

```yaml
cluster:
  name: c2
  masterNodes: 1
  workerNodes: 1
  version: v1.21.1
```

| Field | Maps to |
|---|---|
| `cluster.name` | `metadata.name` on every resource |
| `cluster.masterNodes` | `KubeadmControlPlane.spec.replicas` |
| `cluster.workerNodes` | `MachineDeployment.spec.replicas` |
| `cluster.version` | `KubeadmControlPlane.spec.version` / `MachineDeployment.spec.template.spec.version` |

Scaling a cluster up or down is a one-line change in this file, committed and synced — CAPI
reconciles the difference automatically (creates/deletes `Machine` objects to match).

---

## Bootstrap (one-time, manual)

```bash
# 1. Raise host fd/inotify limits — required for running multiple nested kind/CAPD clusters
#    (see Known Issues — this is the #1 cause of mysterious containerd/kube-proxy failures)
sudo tee /etc/sysctl.d/99-kind.conf <<EOF
fs.file-max=2097152
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
EOF
sudo sysctl --system

# 2. Create the management cluster
kind create cluster --name mgmt

# 3. Initialize CAPI with ClusterTopology and Docker provider
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker

# 4. Install ArgoCD (full manifest — core-install.yaml omits the ApplicationSet CRD)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 5. Bootstrap the app-of-apps — this is the ONLY manifest applied by hand, ever again
kubectl apply -f argocd-apps/root-app.yaml -n argocd
```

From this point on, everything — the ClusterClass/static manifests, and every workload cluster —
is created automatically by ArgoCD from what's committed to this repo.

---

## Installing the CNI (Calico)

Kubeadm clusters have no CNI by default; nodes stay `NotReady` until one is installed.

```bash
clusterctl get kubeconfig c2 > c2.kubeconfig
kubectl --kubeconfig=c2.kubeconfig apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
kubectl --kubeconfig=c2.kubeconfig get nodes -w
```

> Always regenerate the kubeconfig with `clusterctl get kubeconfig`, never reuse the `kind-<cluster>`
> context — see [Known Issues](#known-issues--lessons-learned).

---

## MachineHealthCheck

Each cluster template includes two `MachineHealthCheck` objects (control plane + workers) so unhealthy
nodes are detected and replaced automatically, without manual `kubectl delete machine` intervention.

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineHealthCheck
metadata:
  name: {{ .Values.cluster.name }}-worker-mhc
spec:
  clusterName: {{ .Values.cluster.name }}
  maxUnhealthy: 40%          # ceiling — blocks remediation if too many machines fail AT ONCE
  nodeStartupTimeout: 10m0s  # covers a Machine that never registers a Node at all
  selector:
    matchLabels:
      cluster.x-k8s.io/deployment-name: {{ .Values.cluster.name }}-md-0
  unhealthyConditions:
    - type: Ready
      status: Unknown
      timeout: 5m0s           # covers a Node that exists but stays NotReady too long
    - type: Ready
      status: "False"
      timeout: 5m0s
```

**Key tuning notes:**
- `timeout` must be long enough to cover normal Calico/CNI bring-up on a brand-new node
  (~1-3 min) — too short causes a remediation loop that kills nodes mid-startup.
- `maxUnhealthy` is a **ceiling**, not a floor: it blocks remediation once *more* than this
  percentage is unhealthy at once (protects against acting on a false alarm like a network
  partition). It does not require reaching the threshold before remediation starts — even 1
  unhealthy machine out of many is remediated as long as it's under the ceiling.
- This project pins `apiVersion: cluster.x-k8s.io/v1beta1`. The management cluster's CAPI install
  only serves this version — using `v1beta2` field names (`checks:`, `remediation.triggerIf`)
  under a `v1beta1` `apiVersion` is silently ignored by the API server.

**Testing remediation:**
```bash
docker stop <worker-container-name>
kubectl get machinehealthcheck -w   # CURRENTHEALTHY drops, then a replacement Machine appears
```

---

## Known Issues / Lessons Learned

These are real problems hit and fixed during development of this repo — documented here so they
aren't re-debugged from scratch.

### `Failed to allocate manager object` / `too many open files` (root cause of most instability)
Running several nested `kind`/CAPD clusters on one host exhausts the default, low file-descriptor
and inotify limits. This surfaces as seemingly random failures across every layer: containerd
failing to start, kube-proxy crash-looping, Calico's `install-cni` timing out. **Fix:** raise
`fs.file-max`, `fs.inotify.max_user_instances`, `fs.inotify.max_user_watches` (see Bootstrap step 1).
This single fix resolved the majority of issues encountered.

### `DockerMachineTemplate` / `*Template` objects are immutable
`spec.template.spec` on any `*Template` kind cannot be edited in place — CAPI rejects the patch.
To change a template's spec (e.g. adding `extraMounts`), **rename** the object (e.g. `-v2` suffix)
so it's created fresh, and update every reference to the old name.

### Missing `extraMounts` on worker `DockerMachineTemplate`
CAPD nodes need `/var/run/docker.sock` bind-mounted for Docker-in-Docker mechanics. Missing this
on the **worker** template (while present on the control-plane one) causes containerd to fail to
start on workers only, while the control plane boots fine — a subtle, easy-to-miss asymmetry.

### `controlPlaneEndpoint` drift after manual container intervention
`Cluster.spec.controlPlaneEndpoint` (the load-balancer IP) is set once at cluster creation and
**never reconciled** if the underlying `<cluster>-lb` container is later recreated (e.g. after a
Docker/host restart, or manual `docker rm`/`docker stop`+`start`). This causes
`connection to the workload cluster is down` errors in CAPI's own controllers — not just kubectl
access. **There is no safe live fix.** The only reliable resolution is a full `kubectl delete
cluster <name>` + clean recreate.

### Stale `kind-<cluster>` kubeconfig context
`kind`'s auto-generated context caches a specific host:port for the API server. Since CAPD load
balancer containers get a new Docker-assigned port/IP on every recreation, this context goes stale.
**Always** regenerate access via `clusterctl get kubeconfig <cluster> > <cluster>.kubeconfig`
rather than `kubectl config use-context kind-<cluster>`.

### `kubectl apply` on large CRDs (e.g. ApplicationSet)
`kubectl apply` stores the full manifest in a `last-applied-configuration` annotation, capped at
256KB — some CRDs exceed this. Use `--server-side` instead, which doesn't use that annotation.

### Stale/reused container volumes cause `FileAvailable` preflight errors
A recreated `Machine` that reuses a Docker volume with leftover `/etc/kubernetes/*` files fails
kubeadm's preflight checks (`kubelet.conf`/`ca.crt` already exists). Delete both the `Machine` and
its underlying container + named volume together, not just the `Machine` object.

---

## Known Limitations of the Docker Provider

CAPD (Docker-in-Docker nodes) is intended for **testing CAPI itself**, not for running real
workloads. Compared to a real cloud provider (AWS/GCP/Azure), it has structural weaknesses this
repo works around but cannot fully eliminate:

- Nested containers share the host kernel — resource exhaustion on the host cascades into every
  cluster running on it.
- Node "IPs" are Docker bridge-network addresses, reassigned on every container restart — unlike
  a cloud load balancer's stable DNS/static IP.
- Concurrent node container creation can race on containerd startup under host load.

For a from-scratch discussion of moving to a real cloud infrastructure provider (AWS/GCP/Azure),
see the CAPI docs for the relevant `cluster-api-provider-*` project — the higher-level API
(`Cluster`, `MachineDeployment`, `MachineHealthCheck`) is unchanged; only the infrastructure
provider CRDs (`AWSCluster`, `GCPMachineTemplate`, etc.) differ.

---

## Useful Commands

```bash
# Cluster status
clusterctl describe cluster c2

# Get/refresh a workload cluster's kubeconfig (always use this, never kind-<cluster>)
clusterctl get kubeconfig c2 > c2.kubeconfig

# Watch machine provisioning
kubectl get machines -w

# Force-clear a finalizer-wedged object (last resort)
kubectl patch <kind> <name> -p '{"metadata":{"finalizers":[]}}' --type=merge
```