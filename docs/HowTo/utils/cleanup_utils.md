# Clean Up Utils

## Overview

The Utils domain provides separate cleanup workflows for cluster-log
collection, unattended operating-system installation, and OIM log backups.
Use the scoped tags when only one utility must be cleaned. The general
`cleanup` tag runs log-collection and OS-installation cleanup only.

!!! warning

    Utils cleanup deletes artifacts. Copy any required support or OIM log
    backups before cleaning them, and confirm whether installation credentials
    must be preserved before cleaning the OS-install workflow.

## Prerequisites

- Complete the [OIM setup](../main/setup_oim.md).
- Initialize the Utils domain with `./omnia.sh -i utils`.
- Confirm `OMNIA_DATA_PATH` and `OMNIA_PROJECT_NAME` select the intended
  project.
- Copy log archives that must be retained out of the Utils project output.

## Procedure

To clean the cluster-log collection and unattended OS-installation workflows,
run:

```bash title="Run from: <omnia-repository>/src/main"
./omnia.sh --run utils --tags cleanup
```

!!! note

    The `cleanup` tag does not remove OIM log backups. Use
    `cleanup_backup_oim_logs` explicitly when those backups must be removed.

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

To remove OIM log backups, run:

```bash title="Run from: <omnia-repository>/src/main"
./omnia.sh --run utils --tags cleanup_backup_oim_logs
```

The workflow resolves the destination with the same priority used by
`backup_oim_logs`: command-line `backup_path`, configuration-file
`backup_path`, `OMNIA_BACKUP_PATH`, and then the default project output path.
For a custom or NFS destination, supply or configure the same path used when
the backups were created.

!!! danger

    `cleanup_backup_oim_logs` removes every directory matching
    `omnia_oim_logs_*` at the resolved destination. It does not apply a
    retention period or ask for confirmation. Preserve required backups before
    running it.

## Verification

Review the domain status and the applicable artifact locations:

```bash title="Run on: OIM"
cat "$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/utils_status.yml"
find "$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/collect" \
  -maxdepth 2 -type f 2>/dev/null
find "$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/backup_oim_logs" \
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
- [Back up OIM logs](backup_oim_logs.md) again after reviewing the selected
  domains and destination.

## Troubleshooting

- **A required log bundle was removed**: The cleanup does not retain run
  directories based on archive age. Restore the bundle from the external copy.
- **The NFS mount remains active**: Check `mountpoint /tmp/install_os_nfs`,
  unmount it safely, and rerun the scoped cleanup.
- **OIM log backups remain after cleanup**: The general `cleanup` tag does not
  include them. Run `cleanup_backup_oim_logs` with the same destination that
  was used to create the backups.
- **Credentials are requested on the next installation**: The encrypted
  credential file was removed. Run the installation interactively and provide
  the BMC username, BMC password, and OS root password again.
- **The Vault key remains after credential cleanup**: Remove
  `.install_os_credentials_key` securely from the active Utils project input
  directory after confirming that the encrypted credential file is no longer
  needed.
