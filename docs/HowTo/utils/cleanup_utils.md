# Clean Up Utils

## Overview

The Utils domain provides separate cleanup workflows for cluster-log
collection and unattended operating-system installation. Use the scoped tags
when only one utility must be cleaned, or use `cleanup` to run both workflows.

!!! warning

    Utils cleanup deletes local artifacts. Copy any required support bundles
    before cleaning logs, and confirm whether installation credentials must be
    preserved before cleaning the OS-install workflow.

## Prerequisites

- Complete the [OIM setup](../main/setup_oim.md).
- Initialize the Utils domain with `./omnia.sh -i utils`.
- Confirm `OMNIA_DATA_PATH` and `OMNIA_PROJECT_NAME` select the intended
  project.
- Copy log archives that must be retained out of the Utils project output.

## Procedure

To clean all Utils workflows, run:

```bash title="Run from: <omnia-repository>/src/main"
./omnia.sh --run utils --tags cleanup
```

To clean only log-collection artifacts, run:

```bash title="Run from: <omnia-repository>/src/main"
./omnia.sh --run utils --tags cleanup_logs
```

The log cleanup checks for `omnia_logs_*.tar.gz` archives older than seven days
by default. It then removes every `omnia_logs_*` run directory and the
temporary `k8s` and `slurm` workspaces under:

```text
$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/collect/
```

Because each archive and its `metadata.json` are stored inside a run directory,
copy required files elsewhere before running this command.

To clean only unattended-installation artifacts, run:

```bash title="Run from: <omnia-repository>/src/main"
./omnia.sh --run utils --tags cleanup_install_os
```

This workflow removes `/tmp/install_os`, unmounts `/tmp/install_os_nfs` when it
is mounted, and removes the temporary NFS mount point. When a stored credential
file exists, the playbook asks whether it should be deleted. For a
non-interactive decision, pass one of the following values:

```bash title="Run from: <omnia-repository>/src/main"
./omnia.sh --run utils --tags cleanup_install_os -e cleanup_credentials=true
./omnia.sh --run utils --tags cleanup_install_os -e cleanup_credentials=false
```

!!! important

    The current credential role stores its Vault key as
    `.install_os_credentials_key`, but the cleanup playbook targets the legacy
    name `.install_os_vault_key`. If credentials are being reset, verify the
    active project input directory and securely remove the remaining
    `.install_os_credentials_key` after the workflow completes.

## Verification

Review the domain status and the applicable artifact locations:

```bash title="Run on: OIM"
cat "$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/utils_status.yml"
find "$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/collect" \
  -maxdepth 2 -type f 2>/dev/null
find "$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME" \
  -maxdepth 1 \( -name 'install_os_credentials.yml' \
  -o -name '.install_os_credentials_key' \)
```

Confirm that required support bundles were preserved and that credential files
match the choice made during cleanup.

## Next steps

- [Collect cluster logs](../../Operations/collect_cluster_logs.md) again after
  correcting the condition under investigation.
- [Install an OS unattended](install_os_unattended.md) again after reviewing
  the target BMC, administrative address, and installation disk.

## Troubleshooting

- **A required log bundle was removed**: The cleanup does not retain run
  directories based on archive age. Restore the bundle from the external copy.
- **The NFS mount remains active**: Check `mountpoint /tmp/install_os_nfs`,
  unmount it safely, and rerun the scoped cleanup.
- **Credentials are requested on the next installation**: The encrypted
  credential file was removed. Run the installation interactively and provide
  the BMC username, BMC password, and OS root password again.
- **The Vault key remains after credential cleanup**: Remove
  `.install_os_credentials_key` securely from the active Utils project input
  directory after confirming that the encrypted credential file is no longer
  needed.
