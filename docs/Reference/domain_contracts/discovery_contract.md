# Discovery Input/Output Contract

**Deployment module**: Discovery | **CLI identifier**: `discovery` | **Collection**: `omnia.discovery`

## Input contract

Discovery reads project-scoped inputs from:

```text
$OMNIA_DATA_PATH/discovery/input/$OMNIA_PROJECT_NAME/
```

The defaults are `/opt/omnia` and `project_default`.

| Input | Required | Purpose |
|---|---|---|
| `discovery_config.yml` | Yes | Enables or disables OME discovery and identifies the OME appliance. |
| `network_spec.yml` | Yes during discovery execution | Supplies the admin and InfiniBand subnets used to derive node addresses. |
| `discovery_credentials.yml` | When OME discovery is enabled | Ansible Vault-encrypted OME username and password, created by the credential workflow. |
| `.discovery_credentials_key` | With the credential file | Vault password file used by the Discovery roles. |

### `discovery_config.yml`

The schema is
`plugins/module_utils/discovery_validation/schema/discovery_config.json`.

| Field | Type | Required | Purpose |
|---|---|---|---|
| `enable_bmc_discovery` | boolean | Yes | Set to `true` to run discovery through Dell OpenManage Enterprise (OME). |
| `ome_ip` | IPv4 string | Yes | OME address. It must be a valid, non-loopback IPv4 address when discovery is enabled. |

The current executable discovery flow supports OME. The Magellan section in
the source template is reserved for future use; a manual inventory is supplied
directly to Orchestrator as `pxe_mapping_file.csv` rather than executed as a
Discovery mechanism.

## Output contract

Discovery writes its customer-readable artifacts to:

```text
$OMNIA_DATA_PATH/discovery/output/$OMNIA_PROJECT_NAME/
```

| Output | Purpose |
|---|---|
| `bmc_pxe_mapping_file_<timestamp>.csv` | Timestamped mapping generated from OME inventory. |
| `bmc_pxe_mapping_file.csv` | Symbolic link to the latest timestamped mapping. |
| `bmc_discovery_report_<timestamp>.csv` | Informational server and NIC discovery report. |
| `discovery_status.yml` | Machine-readable execution result. |

### Mapping-file schema

The generated CSV uses these columns:

```text
FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
```

For OME discovery, `FUNCTIONAL_GROUP_NAME` is derived from the server's
supported, case-sensitive OME static-group name. A server without a static
group uses `slurm_node_aarch64`; a server in an unsupported nonempty group is
omitted from the mapping file; and membership in multiple processed groups
causes Discovery to stop. See [Create OME static
groups](../../HowTo/discovery/discover_nodes.md#create-ome-static-groups) for
the supported names and OME procedure.

Review and correct the generated values before copying the file to:

```text
$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/pxe_mapping_file.csv
```

### `discovery_status.yml`

| Field | Purpose |
|---|---|
| `overall_status` | `success` or `failed`. |
| `discovery_mechanism` | Records `ome` for the current executable flow. |
| `bmc_pxe_mapping_file` | Absolute path to the timestamped mapping output. |
| `servers_discovered` | Number of servers returned by discovery. |
| `timestamp` | Execution timestamp. |
| `failed_task` | Present after a failed execution. |
| `failure_reason` | Present after a failed execution. |

## Related documentation

- [Discovery](../../HowTo/discovery/index.md)
- [Discover nodes using OME](../../HowTo/discovery/discover_nodes.md)
- [Discovery configuration](../Configuration/discovery_config.md)
- [PXE mapping file](../SampleFiles/pxe_mapping_file.md)
