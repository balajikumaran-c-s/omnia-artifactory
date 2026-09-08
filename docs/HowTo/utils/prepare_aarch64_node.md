# Prepare an aarch64 Node for Image Building

## Overview

The Utils OS installation workflow supports `aarch64` through the same
`install_os.yml` playbook used for `x86_64`. This guide describes only the
architecture-specific choices. Follow
[Install an OS unattended](install_os_unattended.md) for the complete input,
credential, build, deployment, and verification procedure.

For `aarch64`, the generated GRUB configuration uses `linux` and `initrd`.
For `x86_64`, it uses `linuxefi` and `initrdefi`.

## Prerequisites

- Meet all prerequisites in
  [Install an OS unattended](install_os_unattended.md#prerequisites).
- Make an `aarch64` RHEL 10 installation ISO accessible to the OIM.
- Confirm the installation disk and network-interface names on the target
  hardware.
- Confirm that the target server uses the `aarch64` architecture.

!!! warning

    Kickstart clears and repartitions `install_disk`. Confirm the device name
    on the target server before starting the workflow.

## Procedure

1. Configure all common installation settings as described in
   [Install an OS unattended](install_os_unattended.md#procedure).

2. In
   `$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/install_os_config.yml`,
   use an `aarch64` ISO and set the target architecture:

    ```yaml title="install_os_config.yml"
    source_iso_path: "/path/to/RHEL-10.0-aarch64-dvd.iso"
    target_architecture: "aarch64"
    network_device: "<target_interface>"
    install_disk: "<target_disk>"
    ```

    Replace the example path and placeholders with values verified on the
    target. If `target_architecture` is empty, the validator searches the
    source ISO filename for `x86_64` or `aarch64` and otherwise defaults to
    `x86_64`.

3. Run the complete Utils installation workflow:

    ```bash title="Run from: <omnia-repository>/src/utils"
    ansible-playbook playbooks/utils.yml --tags install_os
    ```

Set `rebuild_iso: true` when an existing custom ISO was created with different
architecture, network, disk, password, or SSH-key content.

## Verification

Connect to the installed node and confirm its architecture:

```bash title="Run on: OIM"
ssh root@<target_admin_ip> uname -m
```

The command must return:

```text
aarch64
```

Then inspect the installation result:

```bash title="Run on: OIM"
cat "$OMNIA_DATA_PATH/utils/output/$OMNIA_PROJECT_NAME/install_os_status.yml"
```

The `architecture` field must be `aarch64`. When SSH verification is enabled
and succeeds, `ssh_verified` is `true`.

## Next steps

Use [Build OS Images](../image_build_manager/build_images.md) to configure the
Image Build Manager and build the required `aarch64` functional-group images.

## Troubleshooting

- **Architecture validation fails**: Set `target_architecture: "aarch64"` and
  confirm that `source_iso_path` references an `aarch64` ISO.
- **The wrong boot commands are generated**: Inspect the `architecture` field
  in `install_os_manifest.yml` and rebuild the ISO with
  `rebuild_iso: true` after correcting the architecture.
- **Networking or storage is not detected correctly**: Verify
  `network_device` and `install_disk` against the target hardware. These
  values are deployment-specific.
- **The node does not become reachable**: Check the iDRAC job and the target
  network configuration. The common installation guide documents the SSH
  retry settings.
