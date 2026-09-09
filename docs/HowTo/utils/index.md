# Utilities

## Overview

The `omnia.utils` collection provides optional utilities that run from the
Omnia Infrastructure Manager (OIM). The current Utils entry point supports
installing RHEL on a bare-metal node through iDRAC Virtual Media and cleaning
up artifacts from that workflow.

The OS installation workflow supports both `x86_64` and `aarch64`. The
collection also contains reusable Slurm configuration backup, cleanup, and
rollback roles, but those roles are not exposed by the Utils entry-point
playbook.

## Prerequisites

| Requirement | Supported by the Utils source |
|---|---|
| Operating system | RHEL 10.x or a compatible Enterprise Linux 10 system |
| Python | 3.12 or later |
| Ansible | `ansible-core` 2.20 or later |
| Runtime location | Omnia Infrastructure Manager |
| Environment | `/etc/omnia/omnia.env` installed and consistent with the OIM |
| Project inputs | Initialized under `$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/` |

The default data path is `/opt/omnia`, and the default project name is
`project_default`. Network access, storage, credentials, and target-system
requirements depend on the selected utility.

## Choose a task

| Task | Use it to |
|---|---|
| [Install an OS unattended](install_os_unattended.md) | Build a Kickstart-enabled ISO, attach it through iDRAC Virtual Media, and install one `x86_64` or `aarch64` node. |
| [Use the Slurm configuration roles](../../Operations/slurm_configuration_roles.md) | Integrate the standalone Slurm backup, cleanup, and rollback roles into an administrator-maintained playbook. |

## Contract reference

See the [Utils Input/Output Contract](../../Reference/domain_contracts/utils_contract.md)
for the environment, input files, credentials, output paths, and generated
status structures used by the current workflows.
