# Configure PXE Boot

## Overview

Orchestrator uses Dell iDRAC to set mapped servers to a PXE-compatible boot
target, restart them, and optionally verify that each node registered after
the current PXE operation. The workflow reads nodes from
`pxe_mapping_file.csv`; it does not use a separate Ansible inventory.

By default, `orchestrator_config.yml` enables PXE boot. The optional
`set_pxe_boot_config.yml` file controls restart behavior, the boot-source
override, and node-registration timing.

!!! caution

    The PXE workflow can restart running servers. Stop workloads and save
    required data before you run it.

## Prerequisites

- Complete [Provision Nodes](provision_nodes.md) so boot and cloud-init
  configurations exist in OpenCHAMI.
- Ensure every target row in
  `$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/pxe_mapping_file.csv`
  has `HOSTNAME`, `ADMIN_IP`, and `BMC_IP` values.
- Configure Orchestrator credentials so the encrypted credential file contains
  `bmc_username` and `bmc_password`.
- Ensure the OIM can reach each iDRAC address and each server can reach the OIM
  provisioning network.
- Enable PXE or UEFI HTTP boot in the server firmware and NIC firmware.

## Procedure

1. Confirm that PXE boot is enabled in
   `$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/orchestrator_config.yml`:

    ```yaml
    enable_pxe_boot: true
    ```

2. Optionally edit
   `$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/set_pxe_boot_config.yml`:

    ```yaml
    enable_node_registration: true
    node_registration_pause_minutes: 3
    node_registration_retries: 120
    node_registration_delay: 15
    node_registration_log_pattern: "phone-home"
    restart_host: true
    force_restart: true
    boot_source_override_enabled: continuous
    boot_source_override_target: pxe
    ```

    Set `boot_source_override_target` to `uefi_http` when that is the boot
    method configured on the servers. Set `boot_source_override_enabled` to
    `once` when the override should apply only to the next boot.

    The source still accepts the legacy `enable_phone_home` and
    `phone_home_*` variable names for compatibility, but emits a deprecation
    warning. Use only the `node_registration_*` names in new configurations.

3. Run the Orchestrator PXE workflow from the Omnia source checkout:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run orchestrator --tags pxeboot
    ```

    To retry only selected nodes, provide a CSV with the same mapping columns:

    ```bash title="Run on: OIM"
    ./omnia.sh --run orchestrator --tags pxeboot \
      -e pxeboot_inventory=/path/to/retry_mapping.csv
    ```

## Verification

- Confirm that the play recap reports no failed hosts.
- Review
  `$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/failed_nodes.json`.
  A successful run contains an empty `failed_nodes` list.
- When node-registration verification is enabled, confirm the workflow reports
  that every successfully restarted node is reachable on admin-network TCP
  port 22 and has a boot epoch newer than the start of the PXE operation. The
  workflow derives the boot epoch from `/proc/uptime`. A matching
  metadata-service journal entry is supporting information, not the primary
  success condition.

## Next steps

- [Verify the cluster](../../Operations/verify_cluster.md) after the nodes have
  booted.
- Use [Add Nodes](../../Operations/add_nodes.md) for later additions to the mapping.

## Troubleshooting

- **Credentials are missing**: Run the Orchestrator `credentials` or `prepare`
  workflow, then retry `pxeboot`.
- **No BMC hosts are found**: Confirm that `BMC_IP` is populated in the mapping
  CSV.
- **Node registration times out**: Check admin-network TCP port 22, SSH access,
  `/proc/uptime`, and the OpenCHAMI metadata-service journal. Increase
  `node_registration_retries` or `node_registration_delay` when the hardware
  needs more time to boot. Nodes that fail the iDRAC restart phase are excluded
  from node-registration polling and remain listed in `failed_nodes.json`.
- **iDRAC rejects the boot override**: Confirm the requested boot target is
  enabled in firmware and supported by the installed iDRAC firmware and
  license.
- **A retry should not wait for node registration**: Run the PXE workflow with
  `-e enable_node_registration=false`.
