# Orchestrator Input/Output Contract

**Deployment module**: Orchestrator | **CLI identifier**: `orchestrator` | **Collection**: `omnia.orchestrator`

## Input contract

Orchestrator stages its project inputs under:

```text
$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/
```

The defaults are `/opt/omnia` and `project_default`.

### Project input files

| Input | Requirement | Purpose |
|---|---|---|
| `orchestrator_config.yml` | Required | Selects the mapping and upstream output paths, provisioning language, DHCP lease time, DNS, kernel override, additional cloud-init file, boot parameters, catalog, and PXE-boot behavior. |
| `network_spec.yml` | Required | Defines the admin network, optional relay-served subnets, and optional InfiniBand network. |
| `pxe_mapping_file.csv` | Required unless `pxe_mapping_file_path` selects another file | Defines node identities, functional groups, admin/BMC interfaces, and optional InfiniBand interfaces. |
| `omnia_config.yml` | Required for the selected cluster services | Defines Slurm and service-Kubernetes clusters and their storage references. |
| `storage_config.yml` | Conditional | Defines storage mounts used by the selected clusters. |
| `security_config.yml` | Conditional | Supplies security settings used by enabled services. |
| `high_availability_config.yml` | Required for service Kubernetes | Defines the Kubernetes control-plane virtual IP. |
| `additional_cloud_init.yml` | Conditional | Supplies global or functional-group cloud-init extensions when selected by `additional_cloud_init_config_file`. |
| `set_pxe_boot_config.yml` | Optional for PXE boot | Overrides node-registration timing and PXE-boot settings. |
| `omnia_config_credentials.yml` | Required for credential-consuming flows | Ansible Vault-encrypted provisioning, BMC, Slurm, OpenLDAP, and PowerScale credentials collected for enabled features. |
| `.omnia_config_credentials_key` | With the credential file | Vault password file for the encrypted credentials. |

The source schemas for `orchestrator_config.yml` and `network_spec.yml` are in
`plugins/module_utils/orchestrator_validation/schema/`.

### `orchestrator_config.yml` path fields

| Field | Default |
|---|---|
| `pxe_mapping_file_path` | `$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/pxe_mapping_file.csv` |
| `image_build_manager_output_path` | `$OMNIA_DATA_PATH/image_build_manager/output/$OMNIA_PROJECT_NAME/build_status.yml` |
| `repo_manager_output_path` | `$REPO_MANAGER_DATA_PATH/output/$OMNIA_PROJECT_NAME/repo_status.yml` |
| `catalog_file_path` | `CATALOG_FILE_PATH`, or `$OMNIA_DATA_PATH/catalog/catalog_rhel.json` |

### Upstream contracts

| Producer | Input consumed by Orchestrator | Required contract |
|---|---|---|
| Repo Manager | `repo_status.yml` | Required flows validate `overall_status: success`, `cluster_os_type`, repository mappings, and `repo_manager.certificates.server_crt`; the certificate file must exist. |
| Image Build Manager | `build_status.yml` | Provisioning requires successful functional-group images and S3 endpoint information. |
| Discovery or administrator | `pxe_mapping_file.csv` | Header and node data must follow the PXE mapping contract below. |
| Repo Manager | Catalog JSON | Generates the catalog that supplies OS metadata and selects supported Slurm, Kubernetes, OpenLDAP, UCX, OpenMPI, and other features. Orchestrator uses `catalog_file_path`, `CATALOG_FILE_PATH`, or the default catalog path, in that order. |

### PXE mapping contract

```text
FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
```

`FUNCTIONAL_GROUP_NAME`, `GROUP_NAME`, `SERVICE_TAG`, `HOSTNAME`, `ADMIN_MAC`,
and `ADMIN_IP` identify and group the node. BMC fields are required by physical
PXE-boot operations. `IB_NIC_NAME` and `IB_IP` are supplied together for nodes
that use InfiniBand.

## Output contract

Customer-readable project outputs are written under:

```text
$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/
```

| Output | Purpose |
|---|---|
| `orchestrator_status.yml` | Canonical overall and per-node provisioning or PXE-boot status. |
| `provisioning_report.yml` | Expected and registered node counts, missing nodes, and missing boot or metadata configurations. |
| `orchestrator_inventory.yaml` | Generated Ansible inventory for mapped nodes, including the service Kubernetes VIP when configured. |
| `bmc_group_data.csv` | BMC inventory generated for downstream iDRAC telemetry. |
| `failed_nodes.json` | Per-node failures produced by the iDRAC PXE-boot and registration flow. |
| `pxeboot_status.yml` | Status written when a custom PXE-boot subset is used. |
| `orchestrator_state.yml` | Persisted feature flags used by subsequent and standalone flows. |

Orchestrator also writes the shared generated file:

```text
$OMNIA_DATA_PATH/.data/functional_groups_config.yml
```

This file is derived from `pxe_mapping_file.csv` and is consumed by inventory
generation, OpenCHAMI configuration, Slurm and Kubernetes provisioning, and
validation roles.

### `orchestrator_status.yml`

For provisioning, the status contains `overall_status`, `timestamp`,
`total_nodes`, `success_count`, `failure_count`, and a `nodes` list. Each node
entry contains its xname, status, and failure reason. The PXE-boot flow replaces
the file with its node-registration results when PXE boot is enabled.

### OpenCHAMI runtime artifacts

The active category-based provisioning workflow creates internal files under:

```text
$OMNIA_DATA_PATH/openchami/workdir/nodes/
```

| Runtime artifact | Purpose |
|---|---|
| `nodes_<category>.yaml` | Supplies the category-specific node records registered in SMD. Categories are `kubernetes`, `slurm`, `os`, and `custom`. |
| `hostname_<category>.yaml` | Supplies category-specific xname-to-hostname assignments to metadata-service. |
| `groups-<functional_group>.yml` | Supplies functional-group membership registered in SMD. |
| `groups-common-<name>.yml` | Supplies common metadata-service groups such as SSH, chrony, and node registration. |

These files are internal working data, not customer-readable project outputs.
The generic `nodes.yaml`, `groups.yaml`, and `hostname.yaml` names belong to the
older non-category workflow and are not the primary artifacts generated by the
current provisioning playbooks.

### Service outputs

Orchestrator also creates runtime state through OpenCHAMI and the selected
cluster roles. These include SMD node and group records, BSS boot parameters,
cloud-init configurations, OpenCHAMI services, and the selected Slurm or
Kubernetes deployment. These service resources are not represented by
generated `slurm_config.yml` or `kubernetes_config.yml` files in the project
output directory.

## Related documentation

- [Orchestrator](../../HowTo/orchestrator/index.md)
- [Provision nodes](../../HowTo/orchestrator/provision_nodes.md)
- [Repository Manager contract](repo_manager_contract.md)
- [Image Build Manager contract](image_build_manager_contract.md)
- [PXE mapping file](../SampleFiles/pxe_mapping_file.md)
