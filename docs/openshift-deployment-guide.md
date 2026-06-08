# NVSentinel on OpenShift — Deployment Guide

This fork adapts NVIDIA NVSentinel for OpenShift. The upstream chart works out of the box on vanilla Kubernetes — on OpenShift, several platform-specific adaptations are required. All fixes are codified in the chart and controlled via values override files.

NVSentinel supports two deployment modes:

- **Monitoring only** (default) — monitors GPU health via DCGM and reports status as Kubernetes Node Conditions
- **Full remediation** — adds fault-quarantine (automatic node cordon) and node-drainer (workload eviction) capabilities, backed by a MongoDB datastore

---

## Prerequisites

### Required for all deployments

1. **OpenShift cluster** with GPU-equipped nodes

2. **Node Feature Discovery (NFD) Operator** — install via OperatorHub. Detects GPU hardware and labels nodes.
   ```bash
   oc get pods -n openshift-nfd
   ```

3. **NVIDIA GPU Operator** — install via OperatorHub. Manages GPU drivers, device plugin, and DCGM.
   ```bash
   oc get pods -n nvidia-gpu-operator
   oc get clusterpolicy
   ```

4. **cert-manager** — required for NVSentinel internal TLS:
   ```bash
   helm repo add jetstack https://charts.jetstack.io --force-update
   helm upgrade --install cert-manager jetstack/cert-manager \
     --namespace cert-manager --create-namespace \
     --version v1.19.1 --set installCRDs=true --wait
   ```

5. **Helm 3.0+** and **oc CLI** authenticated to the cluster

### Additional prerequisites for full remediation mode

6. **Percona MongoDB CRDs** — must be installed before Helm can create the MongoDB cluster:
   ```bash
   oc apply --server-side \
     -f distros/kubernetes/nvsentinel/charts/mongodb-store/charts/psmdb-operator/crds/crd.yaml
   ```

7. **Percona registry aliases** — the Percona operator uses Docker Hub short image names that OpenShift rejects. This one-time MachineConfig adds the required aliases (triggers a node reboot, ~5-10 minutes):
   ```bash
   oc apply -f distros/openshift/percona-registry-aliases.yaml

   # Wait for the node to finish updating:
   oc get machineconfigpool master -w
   # Proceed when UPDATED=True, UPDATING=False
   ```

---

## Installation

### Step 1 — Clone and prepare

```bash
git clone git@github.com:ShiraEzra/nvsentinel.git
cd nvsentinel
git checkout ocp-deployment

# Build chart dependencies (one-time)
helm dependency update distros/kubernetes/nvsentinel/
```

### Step 2 — Deploy

Set the desired NVSentinel version:

```bash
NVSENTINEL_VERSION=v1.8.0
```

**Monitoring only:**

```bash
helm upgrade --install nvsentinel \
  ./distros/kubernetes/nvsentinel/ \
  --namespace nvsentinel-ocp \
  --create-namespace \
  --values ./distros/kubernetes/nvsentinel/values-ocp.yaml \
  --set global.image.tag="$NVSENTINEL_VERSION" \
  --timeout 15m
```

**Full remediation (monitoring + fault-quarantine + node-drainer):**

```bash
helm upgrade --install nvsentinel \
  ./distros/kubernetes/nvsentinel/ \
  --namespace nvsentinel-ocp \
  --create-namespace \
  --values ./distros/kubernetes/nvsentinel/values-ocp.yaml \
  --values ./distros/kubernetes/nvsentinel/values-ocp-remediation.yaml \
  --set global.image.tag="$NVSENTINEL_VERSION" \
  --timeout 15m
```

**Full remediation on Single Node OpenShift (SNO):**

Add the following flag to allow all MongoDB replicas on the same node:

```bash
  --set mongodb-store.psmdb-db.replsets.rs0.affinity.antiAffinityTopologyKey=none
```

---

## Verification

### Monitoring mode

```bash
# All pods should be Running with 0 restarts
oc get pods -n nvsentinel-ocp

# Expected pods:
#   gpu-health-monitor-dcgm-4.x-xxxxx   1/1   Running
#   labeler-xxxxx                        1/1   Running
#   platform-connectors-xxxxx            1/1   Running

# Verify DCGM connectivity
oc logs ds/gpu-health-monitor-dcgm-4.x -n nvsentinel-ocp --tail=15
# Look for: "Successfully created DCGM handle"

# Verify health events are flowing
oc logs ds/platform-connectors -n nvsentinel-ocp --tail=15

# Verify GPU health conditions on the node
oc get node <node-name> \
  -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.message}{"\n"}{end}' \
  | grep -i gpu
```

Expected output (`False` = no failure = healthy):

```
GpuDcgmConnectivityFailure  False   No Health Failures
GpuThermalWatch             False   No Health Failures
GpuDriverWatch              False   No Health Failures
GpuMemWatch                 False   No Health Failures
GpuSmWatch                  False   No Health Failures
GpuAllWatch                 False   No Health Failures
```

### Full remediation mode (additional checks)

```bash
# Additional expected pods:
#   fault-quarantine-xxxxx               1/1   Running
#   node-drainer-xxxxx                   1/1   Running
#   mongodb-rs0-{0,1,2}                  2/2   Running
#   nvsentinel-psmdb-operator-xxxxx      1/1   Running
#   create-mongodb-database-xxxxx        0/1   Completed

# Verify fault-quarantine is watching for GPU fault events
oc logs deployment/fault-quarantine -n nvsentinel-ocp --tail=10
# Look for: "Starting event watcher"

# Verify node-drainer is ready
oc logs deployment/node-drainer -n nvsentinel-ocp --tail=10
# Look for: "All components started successfully"
```

---

## What Was Changed From Upstream

All changes are controlled by `openshift.enabled: true`, which is set inside `values-ocp.yaml`. By passing `--values values-ocp.yaml` at install time, all OpenShift fixes are activated. Without this values file, the chart behaves identically to upstream — for vanilla Kubernetes, use the standard upstream install:

```bash
helm install nvsentinel oci://ghcr.io/nvidia/nvsentinel \
  --version "$NVSENTINEL_VERSION" \
  --namespace nvsentinel \
  --create-namespace
```

| # | Issue | Fix | Files |
|---|-------|-----|-------|
| 1 | SCC blocks root pods | ClusterRoleBinding for built-in `privileged` SCC | `templates/openshift-scc.yaml` |
| 2 | SELinux blocks Unix socket | `seLinuxOptions: {type: spc_t}` on affected containers | `templates/daemonset.yaml`, `charts/gpu-health-monitor/templates/daemonset-dcgm-{3,4}.x.yaml` |
| 3 | DCGM endpoint wrong namespace | Override `global.dcgm.service.endpoint` | `values-ocp.yaml` |
| 4 | Bitnami MongoDB init bug on OCP | Switch to Percona MongoDB Operator + full image paths | `values-ocp-remediation.yaml`, `distros/openshift/percona-registry-aliases.yaml` |

All paths relative to `distros/kubernetes/nvsentinel/`.

### Values override files

| File | Purpose |
|------|---------|
| `values-ocp.yaml` | Core OpenShift fixes — SCC, SELinux, DCGM endpoint (monitoring only) |
| `values-ocp-remediation.yaml` | Enables MongoDB + fault-quarantine + node-drainer (add-on) |

### Cluster prerequisites

| File | Purpose | When needed |
|------|---------|-------------|
| `distros/openshift/percona-registry-aliases.yaml` | MachineConfig mapping Percona short image names to docker.io | Full remediation mode only |
