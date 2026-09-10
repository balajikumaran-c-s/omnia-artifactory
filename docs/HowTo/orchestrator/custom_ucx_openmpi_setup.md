# Configure Custom UCX and OpenMPI

## Overview

Orchestrator places manual UCX and OpenMPI installation scripts on
login/compiler nodes during provisioning. The scripts download the catalog
artifacts from Repo Manager, compile them, and install the shared toolchain
under `/hpc_tools/benchmarks/`.

The generated cloud-init also configures the DOCA MPI environment as the
default stack. It does not automatically execute the custom UCX or OpenMPI
compilation scripts.

## Prerequisites

- Select `ucx` and `openmpi` through the catalog used by Repo Manager.
  Orchestrator derives the corresponding support flags from catalog group
  names; it does not consume `software_config.json`.
- Complete Repo Manager and confirm that `repo_status.yml` reports
  `overall_status: success`.
- Include a `login_compiler_node_x86_64` or
  `login_compiler_node_aarch64` in
  `$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/pxe_mapping_file.csv`.
- Configure Slurm shared storage in
  `$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/storage_config.yml`
  and reference it from the Slurm cluster entry in `omnia_config.yml`.
- Ensure the selected catalog packages include the build dependencies needed
  by the source archives.

## Procedure

1. Synchronize the selected UCX and OpenMPI catalog content:

    ```bash title="Run on: OIM host"
    cd src/main
    ./omnia.sh --run repo_manager --tags "precheck,download,status"
    ```

2. Confirm that Repo Manager published the selected tarballs and produced a
   successful status contract:

    ```bash title="Run on: OIM host"
    cat "$REPO_MANAGER_DATA_PATH/output/$OMNIA_PROJECT_NAME/repo_status.yml"
    pulp file distribution list --limit 1000
    ```

3. Provision the Slurm nodes. This generates the shared `/hpc_tools` mount
   configuration and places `install_ucx.sh` and `install_openmpi.sh` under
   `/usr/local/bin/` on login/compiler nodes:

    ```bash title="Run on: OIM host"
    cd src/main
    ./omnia.sh --run orchestrator --tags provision
    ```

4. On a provisioned login/compiler node, verify the shared mount and install
   UCX:

    ```bash title="Run on: login/compiler node"
    mountpoint -q /hpc_tools
    /usr/local/bin/install_ucx.sh
    source /etc/profile.d/ucx.sh
    ucx_info -v
    ```

    The script installs UCX under `/hpc_tools/benchmarks/ucx` and writes
    `/var/log/ucx_installation.log`.

5. Install OpenMPI after UCX:

    ```bash title="Run on: login/compiler node"
    /usr/local/bin/install_openmpi.sh
    source /etc/profile.d/openmpi.sh
    mpirun --version
    mpicc --version
    ```

    The script detects the shared UCX installation and Slurm commands when
    available. It installs OpenMPI under
    `/hpc_tools/benchmarks/openmpi` and writes
    `/var/log/openmpi_installation.log`.

## Verification

On the login/compiler node, confirm that both shared installations and their
environment files exist:

```bash title="Run on: login/compiler node"
test -x /hpc_tools/benchmarks/ucx/bin/ucx_info
test -x /hpc_tools/benchmarks/openmpi/bin/mpirun
test -f /etc/profile.d/ucx.sh
test -f /etc/profile.d/openmpi.sh
source /etc/profile.d/ucx.sh
source /etc/profile.d/openmpi.sh
ucx_info -v
mpirun --version
```

On the OIM, also verify that the provisioning contract succeeded:

```bash title="Run on: OIM host"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/orchestrator_status.yml"
```

## Next steps

- [Set up NVIDIA HPC SDK](setup_nvhpc_sdk.md).
- [Configure Slurm with GPUs](slurm_with_gpu.md).
- [Run HPC benchmarks](run_hpc_benchmarks.md).

## Troubleshooting

- **`/hpc_tools` is not mounted:** Confirm that the Slurm cluster references
  an existing storage entry in `storage_config.yml`, then rerun provisioning.
- **A tarball cannot be downloaded:** Confirm that Repo Manager synchronized
  the `ucx` and `openmpi` catalog entries and that the Pulp certificate and
  endpoint in `repo_status.yml` are current.
- **UCX compilation fails:** Review
  `/var/log/ucx_installation.log` and verify that the catalog-selected image
  contains the required compiler and build packages.
- **OpenMPI does not use UCX:** Verify
  `/hpc_tools/benchmarks/ucx/bin/ucx_info` exists before rerunning
  `install_openmpi.sh`.
- **OpenMPI does not detect Slurm:** Verify that `sinfo` is available and
  Munge is configured before rerunning the installation.
