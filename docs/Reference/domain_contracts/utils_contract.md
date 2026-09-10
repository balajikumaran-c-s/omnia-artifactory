# Utils Input/Output Contract

The Utils module provides log collection and unattended operating
system installation through its `playbooks/utils.yml` entry point. This
contract describes the files and artifacts used by those implemented
workflows.

## Input contract

### Runtime environment

| Variable or file | Default | Purpose |
|---|---|---|
| `OMNIA_DATA_PATH` | `/opt/omnia` | Base directory for Utils runtime data. |
| `OMNIA_PROJECT_NAME` | `project_default` | Selects the project-specific input and output directories. |
| `SYSTEM_ADMIN_NIC_IPV4` | Empty | When set, the `precheck` flow verifies that the address is present on an OIM interface. |
| `SYSTEM_HOSTNAME` | Empty | When set, the `precheck` flow compares it with `hostname -s`. |
| `SYSTEM_DOMAIN_NAME` | Empty | When set and `hostname -d` is available, the `precheck` flow compares the two values. |
| `/etc/omnia/omnia.env` | None | System environment file checked by the `precheck` flow. A missing file produces a warning. |

`domain-init.sh` stages source templates in:

```text
$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/
```

### `install_os_config.yml`

This user-managed file configures ISO creation and delivery through iDRAC
Virtual Media.

| Field | Required for | Default | Purpose |
|---|---|---|---|
| `source_iso_path` | ISO build or Kickstart generation | Empty | Local path to the source installation ISO. |
| `source_iso_checksum` | No | Empty | Optional SHA-256 checksum used to verify the source ISO. |
| `custom_iso_path` | ISO build, Kickstart generation, or deployment | Empty | Custom ISO and generated-artifact location in `nfs_server:/export/path/file.iso` format. |
| `kickstart_delivery_method` | No | `embedded` | Selects `embedded` or `nfs` Kickstart delivery. |
| `kickstart_file` | No | Empty | Optional user-provided Kickstart file. |
| `kickstart_template` | No | `rhel10` | Built-in template used when `kickstart_file` is empty. |
| `target_bmc_ip` | Deployment | Empty | iDRAC address of the target node. |
| `target_hostname` | ISO build or Kickstart generation | Empty | Hostname written into the generated static-network Kickstart configuration. |
| `target_admin_ip` | ISO build, Kickstart generation, or deployment | Empty | Static operating-system address and SSH verification target. |
| `target_architecture` | No | Detected from the ISO filename | Accepts `x86_64` or `aarch64`; set it explicitly when the filename does not contain the architecture. |
| `network_device` | No | First active link | Network device configured by Kickstart. |
| `netmask` | No | `255.255.255.0` | Static network mask. |
| `gateway` | No | Empty | Static default gateway. |
| `dns_server` | No | Empty | DNS server written into the Kickstart network configuration. |
| `ssh_public_key_path` | No | `/root/.ssh/id_rsa.pub` | Public key injected for passwordless root SSH. |
| `install_disk` | No | `sda` | Installation target cleared and repartitioned by Kickstart. |
| `timezone` | No | `UTC` | Installed operating-system timezone. |
| `rebuild_iso` | No | `false` | Rebuilds the custom ISO when it already exists. |
| `force_reinstall` | No | `false` | Allows deployment when the target admin address already accepts SSH. |
| `ssh_verify_enabled` | No | `true` | Enables post-installation SSH verification. |
| `ssh_verify_retries` | No | `60` | Multiplier used with `ssh_verify_delay` to calculate the SSH wait timeout. |
| `ssh_verify_delay` | No | `30` | Initial delay in seconds before checking SSH; also used to calculate the total timeout. |

The credential flow creates an Ansible Vault-encrypted
`install_os_credentials.yml` with `bmc_username`, `bmc_password`, and
`os_root_password`. It stores the Vault password in
`.install_os_credentials_key`. Both files are placed in the active Utils
project input directory and use mode `0600`.

!!! warning

    The current `cleanup_install_os` implementation targets the legacy Vault
    key name `.install_os_vault_key`. After requesting credential cleanup,
    verify whether `.install_os_credentials_key` remains in the project input
    directory and remove it securely when the credentials are intentionally
    being reset.

### `collect_pxe.yml`

This user-managed file maps supported functional-group names to lists of node
administrative IP addresses:

| Key | Collection target |
|---|---|
| `service_kube_control_plane_x86_64` | Kubernetes control-plane nodes |
| `service_kube_node_x86_64` | Kubernetes worker nodes |
| `slurm_control_node_x86_64` | Slurm controller nodes |
| `slurm_node_x86_64` | x86_64 Slurm compute nodes |
| `slurm_node_aarch64` | aarch64 Slurm compute nodes |
| `login_node_x86_64` | Login nodes |
| `login_compiler_node_aarch64` | Login compiler nodes |

Each value is a YAML list. Empty lists skip the corresponding node type.

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
| `cleanup_logs` | Applies log archive retention and removes collection workspaces. |
| `cleanup_install_os` | Removes temporary installation files and optionally credentials. |
| `cleanup` | Runs both cleanup workflows. |
| `upgrade` | Unsupported placeholder; it only prints a message and performs no upgrade. |
| `rollback` | Unsupported placeholder; it only prints a message and performs no rollback. |

The installation playbook also supports the direct stage tags `credentials`,
`generate_ks`, `build_iso`, and `deploy`.

## Related documentation

- [Utils overview](../../HowTo/utils/index.md)
- [Install an OS unattended](../../HowTo/utils/install_os_unattended.md)
- [Clean up Utils](../../HowTo/utils/cleanup_utils.md)
- [Collect cluster logs](../../Operations/collect_cluster_logs.md)
- [Use the Slurm configuration roles](../../Operations/slurm_configuration_roles.md)
