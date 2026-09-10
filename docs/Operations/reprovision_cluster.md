# Re-provision Cluster Nodes

Re-provisioning replaces the diskless image on existing cluster nodes. Nodes
load the operating system from images provided through OpenCHAMI and boot from
the provisioning network. PXE boot is owned by the Orchestrator domain.

!!! warning

    Re-provisioning restarts the selected nodes. Stop workloads, back up
    required data, and confirm the target mapping before continuing.

!!! caution

    Re-provisioning a Slurm control node or Kubernetes control-plane node can
    require the entire corresponding cluster to be re-provisioned.

## Prerequisites

- The OIM and the required OpenCHAMI services are healthy.
- The Omnia environment and required domains are initialized.
- NFS or PowerScale shared storage is accessible from the OIM and the cluster
  nodes.
- The Orchestrator project mapping contains the correct target nodes and BMC
  addresses.
- Dell iDRAC credentials are available for physical-server PXE boot.
- Cluster workloads are stopped or drained before nodes are restarted.

## Re-provision without modifications

If the mapping, catalog, built images, and Orchestrator inputs have not
changed, rerun only the Orchestrator PXE workflow:

```bash title="Run on: OIM"
cd <OMNIA_SOURCE_PATH>/src/main
./omnia.sh --run orchestrator --tags pxeboot
```

The workflow reads
`$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/pxe_mapping_file.csv`.
It does not use the legacy Utils PXE playbook or a separate Ansible inventory.

To re-provision only a reviewed subset of physical nodes, provide a CSV with
the same mapping columns:

```bash title="Run on: OIM"
./omnia.sh --run orchestrator --tags pxeboot \
  -e pxeboot_inventory=/path/to/reprovision_mapping.csv
```

## Re-provision with modifications

Use the following procedure when the mapping, catalog, image configuration, or
Orchestrator inputs have changed.

1. Update the catalog and the appropriate domain project inputs. Update
   `pxe_mapping_file.csv` directly when using mapping-file discovery, or rerun
   Discovery when OME supplies the mapping.

2. If catalog packages or repositories changed, synchronize Repository
   Manager and regenerate its status:

    ```bash title="Run on: OIM"
    cd <OMNIA_SOURCE_PATH>/src/main
    ./omnia.sh --run repo_manager --tags precheck
    ./omnia.sh --run repo_manager --tags download
    ./omnia.sh --run repo_manager --tags status
    ```

3. If the catalog, packages, functional groups, or image settings changed,
   rebuild the configured images:

    ```bash title="Run on: OIM"
    ./omnia.sh --run image_build_manager --tags build
    ```

   Image Build Manager builds the architectures and functional groups selected
   by the current catalog through its domain entry point.

4. Validate the revised Orchestrator inputs and regenerate provisioning,
   boot-service, metadata-service, and inventory content:

    ```bash title="Run on: OIM"
    ./omnia.sh --run orchestrator --tags validate
    ./omnia.sh --run orchestrator --tags provision
    ```

5. PXE boot the reviewed nodes:

    ```bash title="Run on: OIM"
    ./omnia.sh --run orchestrator --tags pxeboot
    ```

For a combined Orchestrator operation, `--tags execute` runs provisioning and
then runs PXE boot when `enable_pxe_boot: true` is configured. The staged
commands above are recommended for maintenance because each phase can be
verified separately.

## NFS Share Cleanup

When a fresh Slurm or Kubernetes cluster will reuse an existing shared-storage
path, clear only the directories owned by that cluster before re-provisioning.
OIM cleanup does not automatically make an arbitrary NFS share safe to reuse.

!!! danger

    Removing shared-storage content is irreversible. Verify the mounted
    filesystem, configured path, cluster ownership, and backup before deleting
    any content. Do not run a recursive deletion command against an unresolved
    variable or an unverified mount point.

### Reuse the same share paths

1. Stop the workloads and power off the affected cluster nodes when required.
2. Identify the exact share paths from the Orchestrator project
   `storage_config.yml`.
3. Back up required data and clear the cluster-owned content using the
   approved storage-administration procedure.
4. Run the Orchestrator `provision` and `pxeboot` workflows.

### Use new share paths

1. Configure new `mounts` entries in
   [storage_config.yml](../Reference/Configuration/storage_config.md).
2. Reference the required storage name from the applicable cluster definition
   in [omnia_config.yml](../Reference/Configuration/omnia_config.md).
3. Run the Orchestrator `validate`, `provision`, and `pxeboot` workflows.

## Verification

Review the Orchestrator outputs:

```bash title="Run on: OIM"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/orchestrator_status.yml"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/provisioning_report.yml"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/failed_nodes.json"
```

When a custom PXE subset was supplied, also review `pxeboot_status.yml` in the
same output directory.

Verify the applicable cluster:

```bash title="Run on: Slurm control node"
sinfo
```

```bash title="Run on: Kubernetes control-plane node"
kubectl get nodes
```

!!! info

    - [Add Nodes](add_nodes.md) or
      [Remove Slurm Compute Nodes](remove_slurm_nodes.md) -- Change the
      supported node inventory without re-imaging retained nodes.
    - [Build Cluster Images](../HowTo/image_build_manager/build_images.md) --
      Build the images selected by the catalog.
    - [Configure Storage](../HowTo/orchestrator/configure_storage.md) -- Manage
      storage configuration for cluster nodes.
    - [Configure PXE Boot](../HowTo/orchestrator/configure_pxe_boot.md) --
      Configure and run the Orchestrator PXE workflow.
