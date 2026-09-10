# Discovery

The Discovery deployment module queries Dell OpenManage Enterprise (OME), generates node
inventory and BMC/PXE mapping artifacts, and hands the reviewed mapping to
Orchestrator. Administrators who do not use OME create the Orchestrator mapping
file directly; manual inventory is not a Discovery execution mechanism.

For the complete configuration, execution, and verification workflow, see
[Discover nodes using OME](discover_nodes.md).

## Overview

The Discovery module (internal identifier: `discovery`; collection:
`omnia.discovery`) discovers hardware through management platforms such as Dell
OpenManage Enterprise. It produces a PXE mapping file that serves as the
primary data contract between the Discovery and Orchestrator modules. It runs
bare-metal on the OIM host with `connection: local`.

## Prerequisites

Before using the Discovery module, ensure the following prerequisites are met:

- **Main setup completed**: The omnia.sh CLI must be installed and configured (`./omnia.sh -s`)
- **OME access**: OpenManage Enterprise must be accessible from the OIM host for automated discovery
- **Network connectivity**: OIM must have network access to BMC/iDRAC interfaces of target servers
- **Input files configured**: `discovery_config.yml` and `network_spec.yml` must be properly configured
- **Credentials prepared**: OME credentials must be available when
  `enable_bmc_discovery` is `true`.
- **iDRAC hostnames planned**: Use consistent physical-location names so
  Discovery can derive the correct node groups. See [Plan iDRAC
  hostnames](discover_nodes.md#plan-idrac-hostnames).
- **OME static groups planned**: Create the required case-sensitive functional
  groups and assign each server to no more than one group. See [Create OME
  static groups](discover_nodes.md#create-ome-static-groups).

## System Context

```
  discovery_config.yml                          bmc_pxe_mapping_file.csv
  network_spec.yml                               bmc_discovery_report_<timestamp>.csv
  discovery_credentials.yml                    +---------------------+
  +---------------------+     +-----------------+ |                     |
  |   Administrator     |---->|   Discovery      |---->|   Orchestrator      |
  |  (input provider)    |     |  (ome_discovery) |     |   (consumer)        |
  +---------------------+     +-----------------+     +---------------------+
                                       |
                                  OME API
                               (inventory query)
```

## When to use this module

- Use when discovering cluster nodes for the first time
- Use when generating PXE mapping files
- Optional when administrators provide a valid Orchestrator PXE mapping file manually
- Use when adding new nodes to the cluster
- Optional third module in the direct deployment sequence (after Image Build Manager)

## Module workflow

The module supports the following execution tags:

| Tag | Description | Credentials | Destructive |
|-----|-------------|-------------|-------------|
| *(none)* | Full discovery flow (validate + credentials + discovery) | Yes | No |
| `validate` | Validate discovery configuration only | No | No |
| `cleanup` | Remove discovery artifacts | No | Yes |

## Execution Flow

1. **Setup** - Set project name, input/output directories, load discovery_config.yml
2. **Validate** - Run module-specific validation (L1 schema + L2 cross-field logic)
3. **Credentials** - Validate credential file existence, prompt for missing OME credentials, encrypt credential files
4. **Discovery** - Validate `enable_bmc_discovery` and `ome_ip`, collect
   inventory through the OME API, generate the PXE mapping CSV, and generate
   the discovery report

## Output Contract

The Discovery module produces the following output contract:

| Output | Location | Purpose |
|--------|----------|---------|
| `bmc_pxe_mapping_file_<timestamp>.csv` | `/opt/omnia/discovery/output/<project>/` | Maps discovered servers to PXE boot parameters for Orchestrator consumption |
| `bmc_pxe_mapping_file.csv` (symlink) | `/opt/omnia/discovery/output/<project>/` | Symbolic link to the latest timestamped mapping file |
| `bmc_discovery_report_<timestamp>.csv` | `/opt/omnia/discovery/output/<project>/` | NIC and inventory report for operator review |
| `discovery_status.yml` | `/opt/omnia/discovery/output/<project>/` | Overall result, mechanism, mapping path, discovered-server count, and failure details when applicable |

## bmc_pxe_mapping_file.csv Structure

The mapping file contains:

- FUNCTIONAL_GROUP_NAME - Node functional group (e.g., slurm_node_aarch64)
- GROUP_NAME - Scalable Unit / group identifier
- SERVICE_TAG - Dell server service tag
- HOSTNAME - Generated hostname using the three-digit `nidxxx` format (e.g., `nid001`)
- ADMIN_MAC, ADMIN_IP - Admin NIC MAC and IP
- BMC_MAC, BMC_IP - BMC/iDRAC information
- IB_NIC_NAME, IB_IP - InfiniBand NIC and IP (if present)

This contract is consumed by:

- **Operator** - For manual review and editing before provisioning
- **Orchestrator** - For PXE boot configuration and node provisioning

## Mapping File Structure

The PXE mapping file generated by discovery contains the following key concepts:

## Groups vs Functional Groups

- **Group** - Based on physical characteristics. Nodes in the same rack or Scalable Unit (SU) are grouped together with a shared `GROUP_NAME`. Groups help with physical organization and management.
- **Functional Group** - Defines what a node does in the cluster. Categorizes nodes by their role (e.g., service_kube_control_plane, slurm_control_node, login_node).

## Column Reference

| Column | Required | Description |
| --- | --- | --- |
| `FUNCTIONAL_GROUP_NAME` | Yes | Node role with architecture suffix (e.g., `slurm_node_x86_64`, `login_node_aarch64`) |
| `GROUP_NAME` | Yes | Physical grouping identifier (e.g., `grp0`, `grp1`). Nodes in the same group must share the same parent |
| `SERVICE_TAG` | Yes | Dell service tag of the server |
| `PARENT_SERVICE_TAG` | No | Service tag of the parent chassis for blade servers. Leave empty for rack servers |
| `HOSTNAME` | Yes | Discovery generates `nid` followed by a three-digit sequence, normally beginning with `nid001`. The supported NID range is `nid000` through `nid999`. When customization is allowed, do not include the domain name. |
| `ADMIN_MAC` | Yes | MAC address of the PXE NIC on the admin network |
| `ADMIN_IP` | Yes | Static IP address on the admin network |
| `BMC_MAC` | Yes | MAC address of the BMC/iDRAC interface |
| `BMC_IP` | Yes | Static IP address on the BMC network |
| `IB_NIC_NAME` | No | FQDD of the InfiniBand NIC port (e.g., `InfiniBand.Slot.7-1`). Leave empty if no IB NIC present |
| `IB_IP` | No | Static IP address on the InfiniBand network |

## Functional Groups

| Functional Group Name | Layer | Description |
| --- | --- | --- |
| `slurm_control_node_x86_64` | Management | Slurm head node. Nodes in this group are configured to run the Slurm controller |
| `slurm_node_x86_64` | Compute | Slurm compute nodes on x86_64 architecture |
| `slurm_node_aarch64` | Compute | Slurm compute nodes on aarch64 architecture |
| `service_kube_control_plane_x86_64` | Management | Kubernetes control plane nodes on the service cluster. HA requires a minimum of 3 nodes |
| `service_kube_node_x86_64` | Management | Kubernetes worker nodes on the service cluster |
| `login_node_x86_64` | Management | User login nodes on x86_64. Handles user login sessions |
| `login_node_aarch64` | Management | User login nodes on aarch64. Handles user login sessions |
| `login_compiler_node_x86_64` | Management | Login and compiler nodes on x86_64. Includes compilation tools |
| `login_compiler_node_aarch64` | Management | Login and compiler nodes on aarch64. Includes compilation tools |
| `os_x86_64` | Compute | Minimal OS baseline for x86_64. Clean environment for downstream platform software |
| `os_aarch64` | Compute | Minimal OS baseline for aarch64. Clean environment for downstream platform software |

## Recommended Software by Functional Groups

| Functional Group Name | Recommended Software |
| --- | --- |
| `service_kube_control_plane_x86_64` | `service_k8s` |
| `service_kube_node_x86_64` | `service_k8s` |
| `slurm_control_node_x86_64` | `slurm_custom`, `openldap`, `ldms` |
| `slurm_node_x86_64` | `slurm_custom`, `openldap`, `ldms` |
| `slurm_node_aarch64` | `slurm_custom`, `openldap`, `ldms` |
| `login_node_x86_64` | `slurm_custom`, `openldap`, `ldms` |
| `login_node_aarch64` | `slurm_custom`, `openldap`, `ldms` |
| `login_compiler_node_x86_64` | `slurm_custom`, `openldap`, `ucx`, `openmpi`, `ldms` |
| `login_compiler_node_aarch64` | `slurm_custom`, `openldap`, `ucx`, `openmpi`, `ldms` |
| `os_x86_64` | `default_packages`, `ldms` |
| `os_aarch64` | `default_packages`, `ldms` |

!!! note

    - At least one functional group is mandatory. You must not change the name of functional groups
    - Each node must be associated with exactly one functional group. Do not assign a node to multiple functional groups
    - Functional group names are **case-sensitive**
    - To set up a service cluster, the `service_kube_node` must be present in the mapping file

## Related Guides

- [Discover Nodes](discover_nodes.md) -- Create and populate OME static groups,
  discover nodes, and generate mapping files
- [Create Mapping File](create_mapping_file.md) -- Manually create PXE mapping files
- [Getting Started: Full Deployment](../../GetStarted/full_deployment.md)
- [Module Contract](../../Reference/domain_contracts/discovery_contract.md)
- [Related Module: Orchestrator](../orchestrator/index.md) -- Consumer of Discovery output
