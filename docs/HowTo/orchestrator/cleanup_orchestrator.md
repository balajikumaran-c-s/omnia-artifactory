# Clean Up Orchestrator

## Overview

Orchestrator provides a full cleanup workflow and a standalone
component-cleanup playbook. Full cleanup removes every enabled Orchestrator
component while preserving the encrypted credential file and Vault key by
default. Credential cleanup is always opt-in.

Component cleanup can remove OpenCHAMI, OpenLDAP, Slurm, Kubernetes, storage
mounts, or generated Orchestrator artifacts independently. Slurm and
Kubernetes cleanup automatically unmounts configured storage before deleting
their managed directories.

!!! danger

    Cleanup is destructive. Slurm and Kubernetes cleanup can permanently
    delete data from mounted shared NFS storage. Back up required data and
    review the selected components before confirming the operation.

## Prerequisites

- Run cleanup from the OIM with the same `OMNIA_DATA_PATH` and
  `OMNIA_PROJECT_NAME` used for deployment.
- Stop cluster workloads that use the selected services or shared storage.
- Back up configuration, application, project, scratch, and other required
  data stored in directories managed through the selected mounts.
- Run customer-facing commands from `src/main`. Run the standalone component
  playbook from `src/orchestrator`.

## Procedure

### Preview full cleanup

Set `DRY_RUN=true` to display the cleanup plan and execute the component tasks
in Ansible check mode:

```bash title="Run on: OIM"
cd src/main
DRY_RUN=true ./omnia.sh --run orchestrator --tags cleanup
```

Review the displayed component order and every shared-storage path before
continuing.

### Clean all enabled components

Run the full cleanup workflow:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run orchestrator --tags cleanup
```

Type exactly `yes` when prompted. Any other response aborts cleanup. This
operation preserves
`$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/omnia_config_credentials.yml`
and `.omnia_config_credentials_key`.

For an approved non-interactive operation, set `SKIP_APPROVAL=true`:

```bash title="Run on: OIM"
SKIP_APPROVAL=true ./omnia.sh --run orchestrator --tags cleanup
```

### Clean credentials

Remove only the Orchestrator credential file and Vault key:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run orchestrator --tags cleanup_credentials
```

Remove all enabled components and credentials in one supported operation:

```bash title="Run on: OIM"
./omnia.sh --run orchestrator --tags cleanup,cleanup_credentials
```

### Clean selected components

Component tags are intentionally rejected by the top-level Orchestrator
playbook. Run the standalone cleanup playbook directly from
`src/orchestrator`:

```bash title="Run on: OIM"
cd src/orchestrator
ansible-playbook playbooks/cleanup/cleanup_orchestrator.yml --tags <component>
```

Supported component tags are:

| Tag | Scope |
|---|---|
| `openchami` | OpenCHAMI services, containers, configuration, and artifacts. |
| `openldap` | OpenLDAP container, Quadlet configuration, and data. |
| `slurm` | Slurm configuration and managed shared-storage directories. It also runs `storage_mounts` first. |
| `k8s` | Kubernetes configuration and managed shared-storage directories. It also runs `storage_mounts` first. |
| `storage_mounts` | Configured NFS unmount and `/etc/fstab` cleanup. |
| `artifacts` | Generated Orchestrator artifacts and state files. |
| `cleanup_credentials` | Encrypted credential file and Vault key. |

Use `DRY_RUN=true` or `SKIP_APPROVAL=true` with the standalone command when
the same preview or explicitly approved non-interactive behavior is required.
Running the standalone playbook without tags selects all enabled components
and preserves credentials.

## Verification

- Confirm that the play recap contains no failed tasks and that the completion
  message lists the intended number of components.
- For a component cleanup, verify only the selected service, configuration,
  mounts, and artifacts were removed.
- When credentials were preserved, confirm both credential files remain in
  the project input directory.
- When `cleanup_credentials` was selected, confirm both credential files were
  removed.

## Next steps

- Run `./omnia.sh --setup-venv` again only when project inputs need to be
  restaged.
- Follow [Deploy OpenCHAMI](deploy_openchami.md) or
  [Provision Nodes](provision_nodes.md) to redeploy the required components.

## Troubleshooting

- **Cleanup aborts immediately:** Run it interactively and type exactly `yes`,
  or set `SKIP_APPROVAL=true` only after reviewing the destructive scope.
- **Shared data was skipped:** Confirm the expected share was mounted on the
  OIM. Cleanup skips server-side data that is not accessible through the
  configured mount.
- **A component tag is rejected:** Run component tags against
  `playbooks/cleanup/cleanup_orchestrator.yml`, not the top-level
  `playbooks/orchestrator.yml`.
- **A mount remains active:** Stop processes using the mount, verify the
  relevant `storage_config.yml` entry, and rerun the `storage_mounts`
  component cleanup.
