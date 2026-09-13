# Configure Kubernetes HA

Configure high availability (HA) for the service Kubernetes control plane
using kube-vip. This page covers the HA architecture, configuration
reference, and troubleshooting.

For the step-by-step deployment procedure including HA configuration, see
[Set Up Service Kubernetes](deploy_kubernetes.md).

## Overview

Omnia deploys **kube-vip** as a static pod on each control-plane node to
provide a floating virtual IP (VIP) for the Kubernetes API server. If the
active control-plane node fails, kube-vip automatically migrates the VIP
to a healthy node, ensuring uninterrupted API access.

!!! important
    The current provisioning path always configures kube-vip from the first
    `service_k8s_cluster_ha` entry. Keep the intended entry first and provide a
    valid `virtual_ip_address`. The `enable_k8s_ha` value is read but does not
    currently disable kube-vip generation.

## Prerequisites

- For an HA topology, define at least three control-plane nodes in the PXE
  mapping using the exact functional-layer name from the selected catalog. For
  the bundled RHEL 10.0 x86_64 catalog, use
  `service_kube_control_plane_rhel_10_0_x86_64`.
- `omnia_config.yml`, `high_availability_config.yml`, and the PXE mapping file
  are staged for the project.
- A virtual IP address is available on the admin network subnet, not assigned to any other device.

## Procedure

Edit `high_availability_config.yml` in the active project's Orchestrator input
directory **before** running provisioning:

```yaml title="File: high_availability_config.yml"
service_k8s_cluster_ha:
  - cluster_name: service_cluster
    enable_k8s_ha: true
    virtual_ip_address: "172.16.107.1"
```

| Parameter | Description |
|---|---|
| `cluster_name` | Identifies the intended cluster. Keep it aligned with the selected entry in [omnia_config.yml](../../Reference/Configuration/omnia_config.md); the current role does not use this field to select an HA entry. |
| `enable_k8s_ha` | Set to `true` for the supported HA configuration. The current role reads this value but does not use it to gate kube-vip generation. |
| `virtual_ip_address` | IPv4 address consumed by the generated kube-vip and Kubernetes API configuration. Reserve a free address on the admin subnet that does not overlap any `ADMIN_IP`, the MetalLB `pod_external_ip_range`, or the OIM admin IP. |

The current input validator does not cross-check the HA cluster name, control-
plane count, VIP subnet, or address conflicts. Verify those conditions before
provisioning.

For the full parameter reference, see
[HA Config Reference](../../Reference/Configuration/high_availability_config.md).

## Verification

After the cluster is provisioned, verify that HA is operational:

1. **Verify the VIP is reachable**:

    ```bash title="Run on: OIM"
    ping -c 3 <virtual_ip_address>
    ```

2. **Check the Kubernetes API via the VIP**:

    ```bash title="Run on: OIM (example)"
    ssh kcp1 'kubectl get nodes'
    ```

    All control-plane and worker nodes should show `Ready`.

3. **Verify kube-vip is running on control-plane nodes**:

    ```bash title="Run on: OIM (example)"
    ssh kcp1 'crictl ps | grep kube-vip'
    ```

## Next steps

- [Set Up Service Kubernetes](deploy_kubernetes.md) -- Deploy the service
  K8s cluster with HA enabled.

## Troubleshooting

### VIP is not reachable after provisioning

Verify that kube-vip is running on the control-plane nodes and the
static pod manifest is present:

```bash title="Run on: OIM (example)"
ssh kcp1 'crictl ps | grep kube-vip'
ssh kcp1 'cat /etc/kubernetes/manifests/kube-vip.yaml'
```

### VIP conflicts with another address or is unreachable

The current input validator does not detect a VIP conflict. If kube-vip cannot
claim the address or the Kubernetes API is unreachable, confirm manually that
`virtual_ip_address` belongs to the admin subnet and does not match any
`ADMIN_IP` in the [PXE mapping file](../../Reference/SampleFiles/pxe_mapping_file.md),
the OIM admin IP, a DHCP range, or an IP within `pod_external_ip_range`.
Choose a different free IP when a conflict exists.

### Common error messages

| Symptom | Cause | Resolution |
|---|---|---|
| Generated configuration contains an empty API endpoint | `virtual_ip_address` is empty | Set a valid IPv4 address in [high_availability_config.yml](../../Reference/Configuration/high_availability_config.md) before provisioning. |
| VIP is unreachable or kube-vip repeatedly restarts | VIP is outside the admin subnet, already in use, or the control-plane interface cannot claim it | Correct the VIP or network configuration, then reprovision the affected Kubernetes nodes. |














