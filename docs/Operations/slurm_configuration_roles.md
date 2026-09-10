# Slurm Configuration Roles

The Utils source tree contains roles for backing up, removing, and
restoring Slurm configuration stored on a shared filesystem.

## Overview

The current utils entry-point playbook does not expose a Slurm configuration
workflow. There is no shipped `slurm_config_util.yml` playbook, and
`./omnia.sh --run utils` has no `config_backup`, `slurm_cleanup`, or
`config_rollback` workflow tag.

The source tree contains the following reusable roles for integration into an
administrator-maintained playbook:

- `slurm_config_backup` at `src/utils/roles/slurm_config_backup`
- `slurm_cleanup` at `src/utils/roles/slurm_cleanup`
- `slurm_config_rollback` at `src/utils/roles/slurm_config_rollback`

!!! important

    Do not use commands that reference `/opt/omnia/utils/slurm_config_util.yml`.
    That playbook is not present in the current source. The `cleanup` and
    `rollback` tags in `playbooks/utils.yml` are not Slurm maintenance actions.

## Prerequisites

Before integrating these roles, ensure that:

- Use an Omnia source checkout and install the Ansible dependencies declared in
  `src/utils/requirements.yml`.
- The play has access to the shared path containing the `slurm/` directory.
- `share_path` identifies that shared path.
- `ctld_list` contains the Slurm controller name used below the shared path.
- For rollback, inventory contains a host named `slurm_controller`, and the OIM
  can manage `slurmdbd` and `slurmctld` on that host.
- A rollback is tested in a non-production environment before it is used on an
  active cluster.

## Procedure

### Backup

The backup role prompts for an optional backup name and creates a timestamped
directory under:

```text
<share_path>/slurm_backups/<name_and_timestamp>/<controller>/
```

It copies these controller directories from `<share_path>/slurm/<controller>/`:

- `etc/slurm`
- `etc/munge`
- `etc/my.cnf.d`

The role currently suppresses copy failures. Review the resulting directory
before treating the backup as recoverable.

### Cleanup

The cleanup role operates on `<share_path>/slurm`. It asks whether to run the
backup role first and then requires the exact confirmation token `YES` before
removing the complete shared Slurm directory.

!!! warning

    This is destructive. The role removes `<share_path>/slurm` recursively; it
    does not perform a partial cleanup.

### Rollback

The rollback role searches `<share_path>/slurm_backups` and displays the newest
20 backups by default. It validates the selected backup, warns about missing
configuration, Munge, or directory content, and asks whether to create a safety
backup before restoring.

The role restores `etc/slurm`, `etc/munge`, and `etc/my.cnf.d` for the first
controller in `ctld_list`. It then:

1. Sets `/etc/slurm/slurmdbd.conf` to mode `0600` when present.
2. Sets `/etc/munge/munge.key` to mode `0400` when present.
3. Restarts `slurmdbd` only when its configuration changed.
4. Confirms that `slurmctld` is running.
5. Runs `scontrol reconfigure` on the `slurm_controller` inventory host.

The role metadata also runs the backup role as a dependency. Expect an
interactive backup-name prompt before rollback processing begins.

## Verification

After a role integration is executed, verify the filesystem content first. For
a backup, confirm that the selected controller directory contains the expected
Slurm and Munge files.

After rollback, run on the Slurm controller:

```bash title="Run on: Slurm controller node"
scontrol ping
sinfo
scontrol show nodes
```

Also confirm that `slurmctld` and Munge remain active before resuming workload
operations.

## Next steps

- Use the standard orchestrator procedures for supported cluster deployment
  and node lifecycle operations.
- Track the addition of a shipped Slurm utility playbook before exposing these
  roles as a routine operator workflow.

## Troubleshooting

- **`slurm_config_util.yml` is not found**: This is expected in the current
  source. No standalone Slurm utility playbook is shipped.
- **No backups are listed**: Confirm that timestamped backup directories exist
  directly below `<share_path>/slurm_backups`.
- **Rollback reports missing files**: Check for
  `<backup>/<controller>/etc/slurm/slurm.conf`; the role will not proceed
  without it.
- **Rollback is applied but reconfiguration fails**: Check `slurmctld` and
  Munge status, review `journalctl -u slurmctld -n 50`, correct configuration
  errors, and run `scontrol reconfigure` again.
