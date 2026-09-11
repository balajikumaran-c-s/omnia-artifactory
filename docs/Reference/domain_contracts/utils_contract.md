# Utils Domain Contract

The Utils module provides cluster-log collection, unattended operating-system
installation, and OIM domain-log backup through its `playbooks/utils.yml`
entry point. This contract describes the files and artifacts used by those
implemented workflows.

## Upstream domain contract

Utils does not require another deployment domain's status output for its
implemented workflows.

## Output contract

### Module status

Every entry-point execution writes:

```text
$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/utils_status.yml
```

The file contains `utility`, `overall_status`, `playbook`, `version`,
`started_at`, and `completed_at`. It can also contain role results, errors,
and warnings when those values are supplied to the status writer.

The current status writer emits `version: "2.2.0"` even though the Utils Galaxy
collection is version `2.3.0`. Treat this field as status-schema metadata until
the source version is aligned; do not use it to determine the installed Omnia
release.

### OS installation artifacts

ISO build operations write these artifacts to the directory selected by
`custom_iso_path`:

| Artifact | Purpose |
|---|---|
| Custom ISO filename from `custom_iso_path` | Installation media attached through iDRAC Virtual Media. |
| `kickstart.ks` | Generated or augmented Kickstart configuration. It is embedded in the ISO or referenced through NFS. |
| `install_os_manifest.yml` | Source and custom ISO details, SHA-256 checksum, size, Kickstart location, build method, architecture, and timestamp. |

Build or deployment runs also write:

```text
$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/install_os_status.yml
```

The installation status contains `utility`, `status`, `timestamp`,
`target_bmc_ip`, `target_admin_ip`, `target_hostname`, `custom_iso_path`,
`architecture`, `kickstart_delivery_method`, and `ssh_verified`.

### Log-collection artifacts

Each `collect` run writes:

```text
$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/collect/
└── omnia_logs_<timestamp>/
    ├── omnia_logs_<timestamp>.tar.gz
    └── metadata.json
```

The archive contains the collected Kubernetes and Slurm log trees. For an
unreachable node or a collection error, the applicable tree contains an
`SSH_COLLECTION_FAILED.txt` marker.

`metadata.json` contains:

- `bundle_name` and `tar_relative_path`
- `tar_sha256`
- UTC and local generation times
- the triggering user and OIM operating system
- the collection identifier and mode
- exclusions applied
- warning count and warning records

The `cleanup_logs` flow first selects `omnia_logs_*.tar.gz` files older than
seven days by default. It then removes every `omnia_logs_*` run directory and
the temporary `k8s` and `slurm` collection directories. Because archives and
metadata are stored inside the run directories, preserve required bundles
before running cleanup.

### OIM log-backup artifacts

`backup_oim_logs` reads the optional project input:

```text
$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/backup_oim_logs_config.yml
```

The `domains` list selects `repo_manager`, `image_build_manager`,
`orchestrator`, `discovery`, `telemetry`, `build_stream`, or `utils`. An empty
list selects all seven. Each source is the domain-level
`$OMNIA_DATA_PATH/<domain>/log` directory.

The destination is selected from command-line `backup_path`, configuration
`backup_path`, `OMNIA_BACKUP_PATH`, or the following default, in that order:

```text
$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/backup_oim_logs/
└── omnia_oim_logs_<timestamp>/
    ├── omnia_oim_logs_<timestamp>.tar.gz
    └── metadata.json
```

The destination can be an absolute local path or a raw NFS export in
`server:/export/path` format. `metadata.json` records included and skipped
domains, UTC and local generation times, the triggering user, OIM operating
system, backup location, exclusions, warnings, and `archive_sha256`.

The workflow skips missing selected domain log directories and fails when no
requested log directory exists. Files ending in `.tmp`, `.temp`, and `.bak`
are excluded.

`cleanup_backup_oim_logs` uses the same destination resolution and removes
all matching `omnia_oim_logs_*` directories. It is not included in the general
Utils `cleanup` tag.

### Standalone Slurm role artifacts

The Slurm configuration roles are not invoked by `playbooks/utils.yml`. When
integrated into an administrator-maintained playbook:

- `slurm_config_backup` copies `etc/slurm`, `etc/munge`, and `etc/my.cnf.d`
  from the first controller in `ctld_list` to
  `<share_path>/slurm_backups/<name_and_timestamp>/<controller>/`.
- `slurm_cleanup` can remove `<share_path>/slurm` after confirmation.
- `slurm_config_rollback` restores a selected backup for the first controller
  in `ctld_list` and reconfigures the Slurm controller.

## Workflow tags

| Tag | Implemented behavior |
|---|---|
| No tag or `setup` | Initializes Utils facts and project paths. |
| `precheck` | Validates the installed environment against the OIM. |
| `collect` | Collects and bundles cluster logs. |
| `install_os` | Runs the complete ISO build and iDRAC deployment workflow. |
| `backup_oim_logs` | Archives selected OIM domain logs to local or NFS storage. |
| `cleanup_logs` | Applies log archive retention and removes collection workspaces. |
| `cleanup_install_os` | Removes temporary installation files and optionally credentials. |
| `cleanup_backup_oim_logs` | Removes all OIM log-backup run directories from the resolved destination. |
| `cleanup` | Runs log-collection and OS-installation cleanup; it excludes OIM log backups. |
| `upgrade` | Unsupported placeholder; it only prints a message and performs no upgrade. |
| `rollback` | Unsupported placeholder; it only prints a message and performs no rollback. |

The installation playbook also supports the direct stage tags `credentials`,
`generate_ks`, `build_iso`, and `deploy`.

## Related documentation

- [Utils overview](../../HowTo/utils/index.md)
- [Install an OS unattended](../../HowTo/utils/install_os_unattended.md)
- [Back up OIM logs](../../HowTo/utils/backup_oim_logs.md)
- [Clean up Utils](../../HowTo/utils/cleanup_utils.md)
- [Collect cluster logs](../../Operations/collect_cluster_logs.md)
- [Use the Slurm configuration roles](../../Operations/slurm_configuration_roles.md)
