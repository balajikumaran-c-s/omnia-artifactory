# Main

## Overview

The Main component provides `omnia.env` and `omnia.sh` for preparing the Omnia
Infrastructure Manager (OIM). Use it to configure the shared environment,
create the Python virtual environment, initialize module dependencies and
stage the catalog samples required by downstream workflows.

After Main setup, continue with Repository Manager. Module execution and
module-specific inputs, credentials, outputs, and verification are documented
in their respective module sections.

## Prerequisites

- Use an Omnia source checkout on the OIM.
- Use Python 3.11 or later. The setup script searches for `python3.12`,
  `python3.11`, and then `python3`.
- Use an account that can write to the configured data and virtual-environment
  paths and to the system paths created during setup.
- Set `SYSTEM_ADMIN_NIC_IPV4` in `src/main/omnia.env` to an IPv4 address
  assigned to an OIM interface.
- Make the package sources required by each module's `requirements.txt` and
  `requirements.yml` files available during dependency installation.

## Choose a task

| Task | Use it to |
|---|---|
| [Configure the environment](configure_environment.md) | Set the required OIM address, shared paths, project, hostname, domain, version, catalog, and optional component path overrides. |
| [Set up the OIM](setup_oim.md) | Install the environment, create the shared virtual environment, initialize modules, and stage catalog samples. |
| [Maintain the Main environment](../../Operations/maintain_main_environment.md) | Audit dependency versions or remove the installed environment while preserving or deleting runtime data. |

## Command reference

Run the source command help for the complete set of currently implemented
options:

```bash title="Run from: <omnia-repository>/src/main"
./omnia.sh --help
```

After setup, continue with the [Repository Manager flow](../repo_manager/index.md).
