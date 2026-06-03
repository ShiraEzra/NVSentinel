# NVSentinel on OpenShift — Deployment Guide

This fork adds OpenShift support to NVIDIA NVSentinel. On vanilla Kubernetes the upstream chart works out of the box. On OpenShift, three things break — this fork fixes all three via a single `values-ocp.yaml` override file and minimal template changes.

## What We Changed (and Why)

### 1. SCC — Pods blocked from running as root

OpenShift's default `restricted-v2` SCC prevents pods from running as `runAsUser: 0` or using `hostPath` volumes. NVSentinel's DaemonSets need both for direct GPU hardware access.

**Fix:** Added `templates/openshift-scc.yaml` — a ClusterRoleBinding that grants the built-in `privileged` SCC to all service accounts in the namespace. Only created when `openshift.enabled: true`.

### 2. SELinux — Unix socket permission denied

CoreOS SELinux blocks creation of the shared Unix socket `/var/run/nvsentinel.sock`, causing platform-connectors to crash with `bind: permission denied`. The gpu-health-monitor also can't read the socket.

**Fix:** Added `seLinuxOptions: {type: spc_t}` to the container security context of platform-connectors and gpu-health-monitor. Conditional on `openshift.enabled`.

### 3. DCGM endpoint — Wrong namespace

The chart defaults to `nvidia-dcgm.gpu-operator.svc:5555`, but on OpenShift the GPU Operator installs into `nvidia-gpu-operator` namespace.

**Fix:** Overridden in `values-ocp.yaml` — no template changes needed, just a values override.

### Changed Files

All paths relative to `distros/kubernetes/nvsentinel/`:

| File | Change |
|------|--------|
| `templates/openshift-scc.yaml` | **New** — ClusterRoleBinding for privileged SCC |
| `values-ocp.yaml` | **New** — OpenShift values override (DCGM endpoint, openshift flag) |
| `values.yaml` | Added `openshift.enabled: false` and `global.openshift.enabled: false` defaults |
| `templates/daemonset.yaml` | Added conditional `seLinuxOptions: {type: spc_t}` |
| `charts/gpu-health-monitor/templates/daemonset-dcgm-4.x.yaml` | Added conditional `seLinuxOptions: {type: spc_t}` |
| `charts/gpu-health-monitor/templates/daemonset-dcgm-3.x.yaml` | Added conditional `seLinuxOptions: {type: spc_t}` |

All fixes are controlled by a single flag: `openshift.enabled: true`. When set to `false` (default), the chart behaves identically to upstream — no OpenShift-specific resources are created.

---

## Prerequisites

1. **OpenShift cluster** with GPU-equipped nodes

2. **Node Feature Discovery (NFD) Operator** — installed via OperatorHub. NFD detects hardware features (such as NVIDIA PCI cards) and labels nodes accordingly. The GPU Operator depends on it.
   ```bash
   oc get pods -n openshift-nfd
   ```

3. **NVIDIA GPU Operator** — installed via OperatorHub. Manages GPU drivers, device plugin, and DCGM. Requires a `ClusterPolicy` custom resource to configure driver deployment.
   ```bash
   oc get pods -n nvidia-gpu-operator
   oc get clusterpolicy
   ```
   All pods should be `Running` or `Completed`.

4. **cert-manager** — required by NVSentinel for internal TLS:
   ```bash
   helm repo add jetstack https://charts.jetstack.io --force-update
   helm upgrade --install cert-manager jetstack/cert-manager \
     --namespace cert-manager --create-namespace \
     --version v1.19.1 --set installCRDs=true --wait
   ```

5. **Helm 3.0+** and **oc CLI** authenticated to the cluster:
   ```bash
   oc whoami
   oc get nodes
   ```

## Installation

```bash
# 1. Clone and checkout the OCP branch
git clone git@github.com:ShiraEzra/nvsentinel.git
cd nvsentinel
git checkout ocp-deployment

# 2. Build chart dependencies (one-time, downloads 20 subcharts)
helm dependency update distros/kubernetes/nvsentinel/

# 3. Install using the local chart with OpenShift values
#    The --values flag passes values-ocp.yaml, which:
#    - Sets openshift.enabled: true (activates SCC + SELinux fixes)
#    - Points DCGM endpoint to nvidia-gpu-operator namespace
#    The --set flag pins the image tag to the desired release version
NVSENTINEL_VERSION=v1.8.0

helm upgrade --install nvsentinel \
  ./distros/kubernetes/nvsentinel/ \
  --namespace nvsentinel-ocp \
  --create-namespace \
  --values ./distros/kubernetes/nvsentinel/values-ocp.yaml \
  --set global.image.tag="$NVSENTINEL_VERSION" \
  --timeout 15m
```

## Verify

```bash
# All pods should be Running with 0 restarts
oc get pods -n nvsentinel-ocp

# Check DCGM connectivity — should see "Successfully created DCGM handle"
oc logs ds/gpu-health-monitor-dcgm-4.x -n nvsentinel-ocp --tail=15

# Check health events are flowing
oc logs ds/platform-connectors -n nvsentinel-ocp --tail=15

# Check GPU health conditions on the node
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
