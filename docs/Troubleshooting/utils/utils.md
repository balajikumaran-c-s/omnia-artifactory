# Utils Issues

Use these resolutions for the current Utils environment, cluster-log
collection, OIM domain-log backup, unattended operating-system installation,
and cleanup workflows. The Utils Ansible log is
`/var/log/omnia/utils/utils.log`.

## Environment precheck fails

???+ note "Symptom"

    `./omnia.sh --run utils --tags precheck` reports a hostname, domain,
    administrative IP, data-path, or environment-file failure.

??? note "Resolution"

    1. Confirm `/etc/omnia/omnia.env` exists and contains the intended OIM
       values.
    2. Compare `SYSTEM_HOSTNAME` with `hostname -s` and
       `SYSTEM_DOMAIN_NAME` with `hostname -d`.
    3. Confirm `SYSTEM_ADMIN_NIC_IPV4` is assigned to an OIM interface.
    4. Confirm `OMNIA_DATA_PATH` exists and is writable.
    5. Rerun `./omnia.sh --setup-venv` only after preserving intentional
       environment customizations.

## Utils input file is missing

???+ note "Symptom"

    `collect_pxe.yml`, `install_os_config.yml`, or
    `backup_oim_logs_config.yml` is not present in the active project input
    directory.

??? note "Resolution"

    1. From `src/main`, run `./omnia.sh -i utils`.
    2. Confirm `OMNIA_DATA_PATH` and `OMNIA_PROJECT_NAME` select the expected
       project.
    3. Edit the staged file under
       `$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/`.

## Log collection is incomplete

???+ note "Symptom"

    The support archive contains warnings or an
    `SSH_COLLECTION_FAILED.txt` marker for one or more nodes.

??? note "Resolution"

    1. Verify that the administrative IP in `collect_pxe.yml` is correct.
    2. Confirm passwordless SSH access from the OIM to the affected node.
    3. Review the warning records in `metadata.json` for unreachable nodes,
       missing sources, or collection errors.
    4. Correct the reported issue and rerun
       `./omnia.sh --run utils --tags collect`.

## OIM log-backup destination fails

???+ note "Symptom"

    `backup_oim_logs` cannot create the destination or mount the configured NFS
    export.

??? note "Resolution"

    1. Confirm that `backup_path` is an absolute local path or uses
       `server:/export/path` NFS syntax.
    2. Confirm that the OIM can resolve and reach the NFS server and that the
       export allows the OIM to mount and write to it.
    3. Verify local directory permissions and available capacity.
    4. Check whether a command-line `backup_path` or `OMNIA_BACKUP_PATH` is
       overriding the value in `backup_oim_logs_config.yml`.

## OIM log-backup domains are skipped

???+ note "Symptom"

    The backup metadata lists one or more requested domains in
    `domains_skipped`, or the workflow reports that no domain log directories
    are available.

??? note "Resolution"

    1. Confirm that each requested domain name is supported.
    2. Verify that `$OMNIA_DATA_PATH/<domain>/log` exists and is readable.
    3. Review `warnings` in `metadata.json` for each skipped source.
    4. Correct the source or domain selection and rerun
       `./omnia.sh --run utils --tags backup_oim_logs`.

## OIM log-backup checksum does not match

???+ note "Symptom"

    The SHA-256 checksum calculated for the archive differs from
    `archive_sha256` in `metadata.json`.

??? note "Resolution"

    Do not use the archive. Check destination health and available capacity,
    remove the incomplete run directory, and create a new backup.

## General Utils cleanup leaves OIM backups

???+ note "Symptom"

    `./omnia.sh --run utils --tags cleanup` completes, but
    `omnia_oim_logs_*` directories remain.

??? note "Resolution"

    This is expected. Preserve all required backups, then run
    `./omnia.sh --run utils --tags cleanup_backup_oim_logs` with the same
    destination resolution used by the backup workflow.

## Source ISO or checksum validation fails

???+ note "Symptom"

    The installation workflow reports that `source_iso_path` does not exist or
    that `source_iso_checksum` does not match.

??? note "Resolution"

    1. Confirm the source ISO is readable from the OIM.
    2. Calculate its SHA-256 checksum and compare it with
       `source_iso_checksum`.
    3. Replace a partial or corrupted ISO before rerunning the workflow.

## Custom ISO NFS path cannot be resolved

???+ note "Symptom"

    ISO build, Kickstart generation, or deployment cannot resolve
    `custom_iso_path` to a local NFS mount.

??? note "Resolution"

    1. Use `server:/export/path/file.iso` format.
    2. Confirm the export is mountable from the OIM.
    3. Confirm the path below the mounted export matches the path in
       `custom_iso_path`.
    4. Confirm the target iDRAC can reach the same NFS server and export.

## SSH public-key validation fails

???+ note "Symptom"

    The installation validator cannot find `ssh_public_key_path`.

??? note "Resolution"

    Create the default key or configure an existing public key:

    ```bash title="Run on: OIM"
    ssh-keygen -t rsa -b 4096 -N "" -f /root/.ssh/id_rsa
    ```

    The default public-key path is `/root/.ssh/id_rsa.pub`.

## Existing node blocks reinstallation

???+ note "Symptom"

    The target administrative IP already accepts SSH and the workflow stops.

??? note "Resolution"

    Keep `force_reinstall: false` when the node must be protected. Set it to
    `true` only after confirming that the selected node and `install_disk` may
    be reimaged.

## Installation does not become reachable

???+ note "Symptom"

    iDRAC starts the installation but SSH verification times out.

??? note "Resolution"

    1. Check the iDRAC virtual console and one-time virtual-CD boot status.
    2. Verify `target_admin_ip`, `network_device`, netmask, gateway, and DNS.
    3. Confirm the injected root public key is correct.
    4. Increase `ssh_verify_retries` or `ssh_verify_delay` when installation
       needs more time.

## Cleanup leaves the installation Vault key

???+ note "Symptom"

    `cleanup_install_os` removes `install_os_credentials.yml`, but
    `.install_os_credentials_key` remains.

??? note "Resolution"

    The current cleanup playbook targets the legacy name
    `.install_os_vault_key`. After confirming that the installation
    credentials are intentionally being reset, securely remove
    `.install_os_credentials_key` from the active Utils project input
    directory.

## Related documentation

- [Utils overview](../../HowTo/utils/index.md)
- [Install an OS unattended](../../HowTo/utils/install_os_unattended.md)
- [Back up OIM logs](../../HowTo/utils/backup_oim_logs.md)
- [Clean up Utils](../../HowTo/utils/cleanup_utils.md)
- [Collect cluster logs](../../Operations/collect_cluster_logs.md)
- [Slurm configuration roles](../../Operations/slurm_configuration_roles.md)
- [Log management](../../Operations/log_management.md)
