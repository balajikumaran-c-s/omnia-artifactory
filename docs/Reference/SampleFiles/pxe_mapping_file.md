# PXE mapping file

The PXE mapping file is the node-inventory contract consumed by Orchestrator.
It assigns each physical server to a group and functional role and supplies
the hostname and network identities used during provisioning.

The default project-scoped location is:

```text
/opt/omnia/orchestrator/input/project_default/pxe_mapping_file.csv
```

Set `pxe_mapping_file_path` in `orchestrator_config.yml` to select another
absolute path.

## Column reference

Retain every column in this header, including optional columns:

```text
FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
```

| Column | Required | Description |
| --- | --- | --- |
| `FUNCTIONAL_GROUP_NAME` | Yes | Functional-layer name from the selected catalog. The value must exactly match the corresponding image name in Image Build Manager output. |
| `GROUP_NAME` | Yes | Scalable Unit or logical group identifier. |
| `SERVICE_TAG` | Yes | Unique Dell server service tag. |
| `PARENT_SERVICE_TAG` | No | For Slurm compute-node roles, the service tag of the service Kubernetes worker in the same group. Leave empty for other roles. |
| `HOSTNAME` | Yes | Unique lowercase hostname without a domain suffix. |
| `ADMIN_MAC` | Yes | Unique MAC address of the admin/PXE NIC. |
| `ADMIN_IP` | Yes | Unique IPv4 address in a configured admin subnet. |
| `BMC_MAC` | No | BMC/iDRAC MAC address. |
| `BMC_IP` | No | BMC/iDRAC IPv4 address. |
| `IB_NIC_NAME` | No | InfiniBand NIC FQDD, such as `InfiniBand.Slot.7-1` or `NIC.InfiniBand.1-3`. |
| `IB_IP` | No | InfiniBand IPv4 address. |

For the default RHEL 10.0 catalog installed by Main, use these exact,
case-sensitive functional-group names:

- `os_rhel_10_0_x86_64`
- `slurm_control_node_rhel_10_0_x86_64`
- `login_node_rhel_10_0_x86_64`
- `service_kube_control_plane_rhel_10_0_x86_64`
- `service_kube_node_rhel_10_0_x86_64`
- `os_rhel_10_0_aarch64`
- `slurm_node_rhel_10_0_aarch64`
- `login_compiler_node_rhel_10_0_aarch64`

Other catalog variants can define different functional layers. Use the exact
`catalog.functionallayer[].name` value from the selected catalog. When using a
Discovery-generated mapping, review and update `FUNCTIONAL_GROUP_NAME` before
passing the file to Orchestrator.

## Sample file

```csv title="pxe_mapping_file.csv"
FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
slurm_control_node_rhel_10_0_x86_64,grp0,ABCD12,,nid001,02:00:00:00:01:01,172.16.107.52,02:00:00:00:02:01,172.17.107.52,InfiniBand.Slot.7-1,192.168.0.100
service_kube_node_rhel_10_0_x86_64,grp1,ABFL82,,nid002,02:00:00:00:01:02,172.16.107.56,02:00:00:00:02:02,172.17.107.56,,
slurm_node_rhel_10_0_aarch64,grp1,ABCD34,ABFL82,nid003,02:00:00:00:01:03,172.16.107.43,02:00:00:00:02:03,172.17.107.43,InfiniBand.Slot.7-2,192.168.0.101
login_compiler_node_rhel_10_0_aarch64,grp8,ABCD78,,nid004,02:00:00:00:01:04,172.16.107.41,02:00:00:00:02:04,172.17.107.41,NIC.InfiniBand.1-1,192.168.0.103
service_kube_control_plane_rhel_10_0_x86_64,grp3,ABFG79,,nid005,02:00:00:00:01:05,172.16.107.53,02:00:00:00:02:05,172.17.107.53,,
os_rhel_10_0_aarch64,grp7,ABEF78,,nid006,02:00:00:00:01:06,172.16.107.61,02:00:00:00:02:06,172.17.107.61,,
```

`ABFL82` is a service Kubernetes worker in `grp1`; it is the parent service
tag for the Slurm compute node in that same group.

## Validation rules

The current Orchestrator input validator checks:

- Presence of the nine required headers from `FUNCTIONAL_GROUP_NAME` through
  `BMC_IP`. Keep the optional InfiniBand columns as part of the full contract.
- Uniqueness of nonempty `SERVICE_TAG`, `HOSTNAME`, and `ADMIN_IP` values.
- IPv4 syntax for nonempty `ADMIN_IP` values.
- Membership of `ADMIN_IP` values in the primary or additional admin subnets
  from the Orchestrator `network_spec.yml`.

Provisioning also consumes the remaining values. Verify service tags, MAC
addresses, BMC addresses, functional groups, parent relationships, and
InfiniBand information against the physical inventory even when initial
validation passes.

When `dns_enabled` is `true`, use the `nidxxx` hostname format, such as
`nid001`. When it is `false`, custom lowercase hostnames are supported. In
both cases, do not include a domain suffix.

## Related documentation

- [Create a mapping file](../../HowTo/discovery/create_mapping_file.md)
- [Discover nodes using OME](../../HowTo/discovery/discover_nodes.md)
- [Orchestrator contract](../domain_contracts/orchestrator_contract.md)
- [Hostname requirements](../Appendices/hostname_requirements.md)
