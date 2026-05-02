# NVIDIA Network Operator (InfiniBand Only)

## Overview

This manifest configures the **NVIDIA Network Operator** to handle **InfiniBand devices only** using RDMA shared devices.

**IMPORTANT**: This is part of a hybrid RDMA architecture:
- **SR-IOV Network Operator** (manifest 25a): Handles RoCE/Ethernet devices with dedicated VFs
- **NVIDIA Network Operator** (this directory): Handles InfiniBand devices with RDMA shared devices

## Why Hybrid Approach?

InfiniBand NICs with many VFs (16+) cause "message too long" errors when SR-IOV operator queries VF state via netlink. The kernel's ~4KB netlink response limit is exceeded.

**Solution:**
- **RoCE devices**: Use SR-IOV (no InfiniBand-specific large responses)
- **InfiniBand devices**: Use RDMA shared devices to avoid netlink queries entirely

## Architecture

### RDMA Shared Devices

Instead of creating VFs, RDMA shared devices allow multiple pods to share the Physical Function (PF):
- No VF creation required
- No netlink PAGE_SIZE limitation
- Separate resource per unique interface name for granular control
- Global resource naming for multi-node deployments

### Per-NIC Resources

**IMPORTANT**: After wave 25b interface normalization, the generator creates one RDMA resource per **unique interface name**:
- `rdma/rdma_shared_nic0` = all `ib_nic0` interfaces (same PCI address across all nodes)
- `rdma/rdma_shared_nic1` = all `ib_nic1` interfaces (same PCI address across all nodes)
- `rdma/rdma_shared_nic2` = all `ib_nic2` interfaces (same PCI address across all nodes)
- ... etc

**Why Interface-Name-Based (not position-based)?**
- Wave 25b normalizes interface names: same PCI address → same interface name (`ib_nicX`)
- Different nodes may have different NICs online (carrier status)
- Interface-name grouping ensures: **same interface name = same physical NIC (PCI address)**
- Position-based grouping would fail: Node A position 0 ≠ Node B position 0 if different NICs are online

**Example:**
- Node A online: `ib_nic0` (0000:18:00.0), `ib_nic3` (0000:40:00.0), `ib_nic5` (0000:5e:00.0)
- Node B online: `ib_nic0` (0000:18:00.0), `ib_nic1` (0000:29:00.0), `ib_nic3` (0000:40:00.0)
- Resource `rdma_shared_nic3` → both nodes' `ib_nic3` (PCI 0000:40:00.0) ✓ Correct!

This allows:
- Pods to request the same resource name regardless of which node they run on
- Scheduler automatically places pods on nodes that have the requested NIC
- Handles heterogeneous configurations (different NICs online per node)

## DOCA Driver Version Override

By default, the OFED/DOCA driver version is auto-discovered from the NVIDIA Network Operator's `ClusterServiceVersion` (CSV). To pin a specific version — or use a custom image/repository — set the environment variables in `00-version-discovery-job.yaml`:

```yaml
env:
- name: OFED_VERSION_OVERRIDE
  value: "doca3.2.2-25.10-2.4.1.0-4"
- name: OFED_IMAGE_OVERRIDE
  value: ""
- name: OFED_REPO_OVERRIDE
  value: ""
```

| Variable | Default (when empty) | Description |
|---|---|---|
| `OFED_VERSION_OVERRIDE` | Auto-discovered from operator CSV | DOCA driver version tag |
| `OFED_IMAGE_OVERRIDE` | `doca-driver` | Image name (not discovered, hardcoded default) |
| `OFED_REPO_OVERRIDE` | `nvcr.io/nvidia/mellanox` | Image repository (not discovered, hardcoded default) |

**Notes:**
- Only the **version** is discovered from the CSV. The image name and repository are always hardcoded defaults unless overridden.
- When `OFED_VERSION_OVERRIDE` is set, the job skips the CSV lookup entirely and uses the provided version.
- After changing any value, sync the ArgoCD app — the job runs as a Sync hook and picks up changes on the next sync.

## Components

### 1. RDMA Configuration Generator (job-generate-rdma-config.yaml)

Auto-generates RDMA configuration from discovered InfiniBand devices:

**Input:**
- IB devices from SR-IOV discovery configmap (`generated-sriov-resources`)
- Only devices with carrier=1 (online, cable connected) are included

**Output:**
- NicClusterPolicy patch with RDMA shared device plugin config (one resource per unique interface name)

**Behavior:**
- Only runs if InfiniBand devices are detected
- If no IB devices found: Only deploys MOFED drivers (no RDMA plugin)
- Groups by interface name (ib_nicX) after wave 25b normalization
- Handles heterogeneous configurations (different NICs online per node)
- Only creates resources for NICs that have carrier=1 on at least one node
- **Pure RDMA mode**: No IP networking (no IPoIB CNI, no NetworkAttachmentDefinitions)

### 2. Base NicClusterPolicy (nicclusterpolicy.yaml)

Empty base policy - actual configuration is generated dynamically by:
1. NIC discovery (manifest 26a): Creates base policy with MOFED drivers
2. RDMA generator (this manifest): Patches in RDMA shared device plugin

**Note:** Pods access InfiniBand devices directly via `/dev/infiniband` hostPath mount. No IP networking is configured for InfiniBand devices.

## Deployment Order

1. **Wave 25a**: SR-IOV Network Operator installation
2. **Wave 26a**: NIC discovery
   - Discovers all Mellanox/NVIDIA NICs
   - Separates RoCE and InfiniBand devices
   - Creates base NicClusterPolicy with MOFED drivers
   - Exports IB devices to configmap
3. **Wave 28**: NVIDIA Network Operator (this directory)
   - Reads IB devices from configmap
   - Patches NicClusterPolicy with RDMA shared device plugin
   - Pure RDMA mode: Pods access devices via hostPath (no IP networking)

## Using InfiniBand RDMA in Pods

**Pure RDMA Mode:** Pods access InfiniBand devices directly via hostPath mount. No network annotations or RDMA resource requests are needed.

### Example Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rdma-test
spec:
  hostNetwork: false
  volumes:
  - name: dev-infiniband
    hostPath:
      path: /dev/infiniband
  containers:
  - name: app
    image: your-rdma-app
    volumeMounts:
    - name: dev-infiniband
      mountPath: /dev/infiniband
    securityContext:
      privileged: true
      capabilities:
        add: ["IPC_LOCK", "SYS_RESOURCE"]
```

**Access InfiniBand devices:**
```bash
# List available devices
ibv_devices

# Use specific device in your RDMA application
ib_write_bw -d mlx5_3 <target-ip>
ucx_perftest -d mlx5_4 -t tag_bw <target-ip>
```

**Result:**
- Pod has direct access to all InfiniBand devices on the node
- No IP configuration needed (uses regular pod IP for connection establishment)
- RDMA data transfer bypasses IP stack completely
- Full bandwidth and lowest latency

## Heterogeneous Configurations

The generator handles nodes with different NIC counts:

**Example Cluster:**
- Node A: 2 NICs connected
- Node B: 8 NICs connected

**Created Resources:**
- `rdma_shared_nic0`: Available on both nodes (both have 1st NIC)
- `rdma_shared_nic1`: Available on both nodes (both have 2nd NIC)
- `rdma_shared_nic2-7`: Only available on Node B

**Scheduler Behavior:**
- Pods requesting `rdma_shared_nic0` or `nic1`: Can schedule on any node
- Pods requesting `rdma_shared_nic2-7`: Only schedule on Node B

## Connectivity Check

Only NICs with physical connectivity (carrier=1) are included in the configuration.

**Discovery checks:**
- `/sys/class/net/<interface>/carrier` must be "1"
- `/sys/class/net/<interface>/operstate` must be "up"

Disconnected or down NICs are automatically excluded.

## Files

- `00-version-discovery-job.yaml`: OFED/DOCA version discovery and NicClusterPolicy creation
  - Auto-discovers driver version from operator CSV, or uses `OFED_VERSION_OVERRIDE`
  - Supports custom image/repo via `OFED_IMAGE_OVERRIDE` / `OFED_REPO_OVERRIDE`
- `job-generate-rdma-config.yaml`: RDMA configuration generator
  - ServiceAccount, ClusterRole, ClusterRoleBinding
  - ConfigMap with Python generator script
  - Job to run generator
- `nicclusterpolicy.yaml`: Base NicClusterPolicy (empty, filled by generators)
- `kustomization.yaml`: Kustomize configuration
- `presync-wait-for-sriov-mcp.yaml`: PreSync hook to wait for MachineConfigs

## Verification

### Check RDMA Resources

```bash
# Check NicClusterPolicy status
oc get nicclusterpolicy nic-cluster-policy -n nvidia-network-operator -o yaml

# Check RDMA resources on nodes
oc get nodes -o json | jq -r '.items[] | .metadata.name as $node | .status.capacity | to_entries[] | select(.key | contains("rdma_shared_nic")) | "\($node): \(.key)=\(.value)"'

# Check RDMA device plugin pods
oc get pods -n nvidia-network-operator -l app=rdma-shared-dp
```

### Check MOFED Drivers

```bash
# Check OFED driver pods
oc get pods -n nvidia-network-operator -l app=mofed-driver

# Check driver version
oc logs -n nvidia-network-operator -l app=mofed-driver -c mofed-container | grep "MOFED version"
```

## Related Manifests

- `25a-sriov-operator`: SR-IOV Network Operator (handles RoCE devices)
- `26a-sriov-discovery`: NIC discovery and base NicClusterPolicy generation
- `26-sriov-vf-config`: VF creation for RoCE devices (MachineConfig)

## References

- [NVIDIA Network Operator Documentation](https://docs.nvidia.com/networking/display/cokan10/network+operator)
- [RDMA Shared Device Plugin](https://github.com/Mellanox/k8s-rdma-shared-dev-plugin)
- [Netlink PAGE_SIZE Issue](https://github.com/k8snetworkplumbingwg/sriov-network-operator/pull/1026)
