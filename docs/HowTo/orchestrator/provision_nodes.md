# Provision Nodes

## Overview

Orchestrator provisions nodes from `pxe_mapping_file.csv`. It classifies each
functional group, registers the corresponding nodes and groups in OpenCHAMI
SMD, creates boot-service and metadata-service configuration, prepares category
bolt-ons, generates inventories, and optionally reboots physical servers
through iDRAC PXE boot.

The source recognizes these categories:

| Functional-group prefix | Provisioning path |
|---|---|
| `service_kube_` | Kubernetes |
| `slurm_` | Slurm |
| `login_node_`, `login_compiler_node_` | Slurm, including login nodes |
| `os_` | OS-only |
| Any other value | Custom |

## Prerequisites

- Complete Repo Manager and Image Build Manager successfully. Orchestrator
  requires a successful `repo_status.yml`, a valid Pulp public certificate, and
  a successful `build_status.yml` containing an image for each functional
  group.
- Provide the discovery mapping with these case-sensitive columns:
  `FUNCTIONAL_GROUP_NAME`, `GROUP_NAME`, `SERVICE_TAG`,
  `PARENT_SERVICE_TAG`, `HOSTNAME`, `ADMIN_MAC`, `ADMIN_IP`, `BMC_MAC`, and
  `BMC_IP`. `IB_NIC_NAME` and `IB_IP` are supported optional columns.
- Use unique service tags, hostnames, and admin IPs. Hostnames must be lowercase,
  must not begin with a number, and must not contain underscores, dots, or
  spaces.
- Configure `network_spec.yml`. Every mapped admin IP must be valid for the
  configured network.
- For physical-server PXE boot, provide reachable Dell iDRAC addresses and BMC
  credentials. For virtual machines or environments without iDRAC, set
  `enable_pxe_boot: false`.
- Configure `omnia_config.yml`, `storage_config.yml`,
  `high_availability_config.yml`, and `security_config.yml` for the catalog
  features used by the mapped functional groups.

## Procedure

### 1. Initialize and configure the module

```bash title="Run on: OIM"
cd src/main
./omnia.sh --setup-venv
```

The input directory is
`$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/`. In
`orchestrator_config.yml`, configure the mapping and any non-default upstream
paths:

```yaml title="orchestrator_config.yml"
pxe_mapping_file_path: "/path/to/pxe_mapping_file.csv"
image_build_manager_output_path: ""
repo_manager_output_path: ""
catalog_file_path: ""
enable_pxe_boot: true
```

Empty upstream paths use the current project defaults. For a mapping already
copied into the project input directory, use its absolute path for
`pxe_mapping_file_path`.

### 2. Validate the inputs and prerequisites

```bash title="Run on: OIM"
./omnia.sh --run orchestrator --tags validate
./omnia.sh --run orchestrator --tags precheck
```

### 3. Run the complete or staged workflow

For a complete run, use the playbook without tags. The default path runs
precheck, prepare, and execute; `execute` includes provisioning and PXE boot
when `enable_pxe_boot` is `true`.

```bash title="Run on: OIM"
./omnia.sh --run orchestrator
```

To control each phase, run one tag at a time:

```bash title="Run on: OIM"
./omnia.sh --run orchestrator --tags credentials
./omnia.sh --run orchestrator --tags deploy
./omnia.sh --run orchestrator --tags provision
./omnia.sh --run orchestrator --tags pxeboot
```

`provision` configures every category present in the mapping and writes a
provisioning report. It does not trigger iDRAC. `pxeboot` sets the boot source,
restarts mapped physical servers, waits for SSH on their admin IPs, verifies
that each boot occurred after the PXE trigger, and writes the final status.

## Verification

Inspect the generated status and provisioning report:

```bash title="Run on: OIM"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/orchestrator_status.yml"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/provisioning_report.yml"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/failed_nodes.json"
```

The provisioning validation compares expected mapping xnames with SMD,
confirms boot-service configurations for functional groups, checks
metadata-service group data and hostname assignments, and generates
`orchestrator_inventory.yaml` and `bmc_group_data.csv` in the same output
directory.

After PXE boot, `orchestrator_status.yml` reports `overall_status`, total,
success, and failure counts, plus each node's `pxe_boot` or
`node_registration` failure stage.

## Next steps

- Use [Deploy Slurm](deploy_slurm.md) or
  [Deploy Kubernetes](deploy_kubernetes.md) to verify the provisioned service.
- Use [Add Nodes](../../Operations/add_nodes.md) to provision a new subset without rebooting
  existing nodes.
- Use [Remove Slurm Nodes](../../Operations/remove_slurm_nodes.md) for source-supported Slurm compute
  node removal.

## Troubleshooting

**Input validation fails for the mapping**

Confirm the configured path, uppercase headers, unique identifiers, valid admin
IPs, and lowercase hostnames. Review the generated validation log under
`/var/log/omnia/orchestrator/orchestrator.log`.

**OpenCHAMI provisioning fails**

The provision phase requires the configuration created by deployment. Check the
services and retry the appropriate phase:

```bash title="Run on: OIM"
systemctl status openchami.target
systemctl status metadata-service
/usr/bin/ochami smd service status
cd src/main
./omnia.sh --run orchestrator --tags deploy
./omnia.sh --run orchestrator --tags provision
```

**PXE boot reports no BMC hosts**

Populate `BMC_IP` in column 9 of every physical-node row. Ensure the OIM can
reach each iDRAC and rerun `--tags pxeboot`.

**Node registration times out**

Check admin-network TCP port 22, metadata-service, and cloud-init on the target
node. The polling window is controlled by `node_registration_pause_minutes`,
`node_registration_retries`, and `node_registration_delay` in
`set_pxe_boot_config.yml`. Failure details are written to `failed_nodes.json`.
