# Unattended OS Installation via iDRAC Virtual Media

## Overview

The utils `install_os` workflow installs RHEL on one bare-metal node through
iDRAC Virtual Media. It validates the installation configuration, collects and
encrypts credentials, creates a custom ISO with Kickstart, attaches that ISO to
the target BMC, power-cycles the server, and optionally verifies SSH access to
the installed operating system.

The same workflow supports `x86_64` and `aarch64`. The target architecture can
be configured explicitly or detected from the source ISO filename.

!!! warning

    The generated Kickstart clears and repartitions the configured
    `install_disk`. Confirm the BMC address, operating-system address, and disk
    name before starting the deployment.

## Prerequisites

- Complete the [OIM setup](../main/setup_oim.md) and initialize the utils
  module.
- Place a RHEL 10 source ISO on a filesystem accessible to the OIM.
- Provide an NFS destination in `server:/export/path/file.iso` format. The OIM
  must be able to mount the export, and the target BMC must be able to access it.
- Ensure the OIM can reach the target BMC through HTTPS and the installed node
  through SSH.
- Ensure the target BMC supports the Redfish operations used by the iDRAC
  virtual-media modules.
- Create the SSH public key configured by `ssh_public_key_path`. The default is
  `/root/.ssh/id_rsa.pub`.

The first installation run prompts for `bmc_username`, `bmc_password`, and
`os_root_password`. The workflow stores them in an Ansible Vault-encrypted
`install_os_credentials.yml` file in the utils project input directory.

## Procedure

### Install the operating system

1. Initialize the Utils module so that its input templates and dependencies are
   available:

    ```bash title="Run from: <omnia-repository>/src/main"
    ./omnia.sh -i utils
    ```

    If the shared Omnia environment has not been created, run `./omnia.sh -s`
    first.

2. Edit the staged configuration:

    ```bash title="Run on: OIM host"
    vi /opt/omnia/utils/input/project_default/install_os_config.yml
    ```

    Replace `project_default` when `OMNIA_PROJECT_NAME` selects another project.

3. Configure the source ISO, NFS destination, and target node. For example:

    ```yaml title="File: <OMNIA_DATA_PATH>/utils/input/<project>/install_os_config.yml"
    source_iso_path: "/opt/omnia/iso/RHEL-10.0-x86_64-dvd.iso"
    source_iso_checksum: ""
    custom_iso_path: "192.0.2.10:/exports/omnia/RHEL-10.0-x86_64-omnia.iso"

    kickstart_delivery_method: embedded
    kickstart_file: ""
    kickstart_template: rhel10

    target_bmc_ip: "192.0.2.21"
    target_hostname: "compute-01"
    target_admin_ip: "192.0.2.31"
    target_architecture: "x86_64"

    network_device: "eno1"
    netmask: "255.255.255.0"
    gateway: "192.0.2.1"
    dns_server: "192.0.2.2"
    ssh_public_key_path: "/root/.ssh/id_rsa.pub"
    install_disk: "sda"
    timezone: "UTC"

    rebuild_iso: false
    force_reinstall: false
    ssh_verify_enabled: true
    ssh_verify_retries: 60
    ssh_verify_delay: 30
    ```

    Replace the example addresses and paths with values for the deployment.

4. Run the complete workflow:

    ```bash title="Run from: <omnia-repository>/src/main"
    ./omnia.sh --run utils --tags install_os
    ```

    Respond to the credential prompts on the first run. Later runs reuse the
    encrypted credential file unless it is removed by the installation cleanup
    workflow.

The workflow performs the following operations:

1. Validates the fields required for ISO build and deployment.
2. Loads or collects the BMC and OS root credentials and injects the OIM public
   key into Kickstart.
3. Verifies the source ISO and its SHA-256 checksum when one is supplied.
4. Installs required ISO tools when they are absent.
5. Creates the custom ISO unless it already exists and `rebuild_iso` is false.
6. Attaches the NFS-hosted ISO through iDRAC Virtual Media and requests a
   one-time virtual-CD boot.
7. Power-cycles the node and, when enabled, waits for SSH to become available.
8. Writes installation and utils status files for the active project.

### Run individual build or deployment stages

For troubleshooting or controlled operation, run the installation playbook
directly from the utils collection after activating the Omnia environment:

```bash title="Run from: <omnia-repository>/src/utils"
ansible-playbook playbooks/install_os.yml --tags credentials
ansible-playbook playbooks/install_os.yml --tags generate_ks
ansible-playbook playbooks/install_os.yml --tags build_iso
ansible-playbook playbooks/install_os.yml --tags deploy
```

`credentials` collects credentials only, `generate_ks` writes the Kickstart
file without building an ISO, `build_iso` builds the custom media, and `deploy`
uses an existing custom ISO.

### `install_os_config.yml` parameter reference

| Parameter | Required | Default | Description |
| --- | --- | --- | --- |
| `source_iso_path` | Build and Kickstart generation | -- | Local path to the source ISO. |
| `source_iso_checksum` | No | Empty | Optional SHA-256 checksum for the source ISO. |
| `custom_iso_path` | Build and deployment | -- | NFS URI for the custom ISO in `server:/path/file.iso` format. |
| `kickstart_delivery_method` | No | `embedded` | Use `embedded` or `nfs` Kickstart delivery. |
| `kickstart_file` | No | Empty | Optional user-provided Kickstart file. Missing root password and SSH-key directives are injected. |
| `kickstart_template` | No | `rhel10` | Built-in Kickstart template name. |
| `target_bmc_ip` | Deployment | -- | Target BMC/iDRAC IP address. |
| `target_hostname` | No | Empty | Hostname written by Kickstart. |
| `target_admin_ip` | Deployment | -- | Static OS address and post-install SSH-verification target. |
| `target_architecture` | No | Detected from ISO name | `x86_64` or `aarch64`. |
| `network_device` | No | First active link | Network interface used by Kickstart. |
| `netmask` | No | `255.255.255.0` | Static network mask. |
| `gateway` | No | Empty | Static default gateway. |
| `dns_server` | No | Empty | DNS server used by Kickstart. |
| `ssh_public_key_path` | No | `/root/.ssh/id_rsa.pub` | Public key injected for root SSH access. |
| `install_disk` | No | `sda` | Disk erased and used for installation. |
| `timezone` | No | `UTC` | Installed-system timezone. |
| `rebuild_iso` | No | `false` | Rebuild an existing custom ISO. |
| `force_reinstall` | No | `false` | Continue when the target OS address already accepts SSH. |
| `ssh_verify_enabled` | No | `true` | Verify SSH after the BMC deployment operation. |
| `ssh_verify_retries` | No | `60` | Multiplier used with `ssh_verify_delay` to calculate the SSH wait timeout. |
| `ssh_verify_delay` | No | `30` | Initial delay in seconds before checking SSH; also used to calculate the total timeout. |

## Verification

1. Confirm that the custom ISO, `kickstart.ks`, and
   `install_os_manifest.yml` exist at the NFS destination directory.

2. Review the generated project status:

    ```bash title="Run on: OIM host"
    cat /opt/omnia/utils/output/project_default/install_os_status.yml
    ```

3. When SSH verification is enabled, connect to the installed node:

    ```bash title="Run on: OIM host"
    ssh root@<target_admin_ip>
    ```

4. Verify the operating system and architecture:

    ```bash title="Run on: target node"
    cat /etc/redhat-release
    uname -m
    ```

## Next steps

- [Build Cluster Images](../image_build_manager/build_images.md) -- Use the
  installed node where required by the image-building workflow.

## Troubleshooting

- **`install_os_config.yml` is not found**: Run `./omnia.sh -i utils` and edit
  the staged file under `<OMNIA_DATA_PATH>/utils/input/<project>/`.
- **The source ISO is rejected**: Confirm `source_iso_path` exists and that
  `source_iso_checksum`, when configured, is the correct SHA-256 value.
- **The custom ISO path cannot be resolved**: Confirm `custom_iso_path` uses
  `server:/path/file.iso` format and that the NFS export is mountable from the
  OIM.
- **The target is already reachable**: Leave `force_reinstall: false` to protect
  an installed node, or set it to `true` only after confirming that the target
  may be reimaged.
- **SSH verification times out**: Verify the configured administrative IP,
  network interface, gateway, and firewall path. Increase
  `ssh_verify_retries` or `ssh_verify_delay` when installation requires longer.
