# Set up the OIM

## Overview

`omnia.sh --setup-venv` installs the Main environment, creates or updates the
shared Python virtual environment, initializes the selected modules, and copies
the supplied catalog samples. Use this command for the initial setup of the
Omnia Infrastructure Manager (OIM).

## Prerequisites

- Use an Omnia source checkout on the OIM.
- Install Python 3.11 or later. Main searches for `python3.12`, `python3.11`,
  and then `python3`.
- Use an account that can write to `/etc/omnia`, `/etc/profile.d`, the
  configured data path, and the configured virtual-environment path.
- Make the Python and Ansible Galaxy package sources declared by the modules
  available during initialization.
- [Configure `src/main/omnia.env`](configure_environment.md). In particular:

    - `SYSTEM_ADMIN_NIC_IPV4` must be an IPv4 address assigned to the OIM.
    - `SYSTEM_HOSTNAME` must match `hostname -s`.

## Procedure

1. Change to the Main source directory:

    ```bash title="Run on: OIM host"
    cd src/main
    ```

2. Run setup:

    ```bash title="Run on: OIM host"
    ./omnia.sh --setup-venv
    ```

    The short form is `./omnia.sh -s`. The command:

    - Copies `omnia.env` to `/etc/omnia/omnia.env`.
    - Creates `/etc/profile.d/omnia-env.sh`.
    - Validates the OIM hostname, domain name, and administrative NIC address.
    - Creates `<OMNIA_DATA_PATH>`, `<OMNIA_DATA_PATH>/.data`, and the virtual
      environment at `OMNIA_VENV_PATH`.
    - Upgrades `pip`, `setuptools`, and `wheel` in the virtual environment.
    - Runs each selected module's `domain-init.sh`.
    - Copies JSON and YAML samples from `src/main/samples/` to
      `<OMNIA_DATA_PATH>/catalog/`.

3. Use setup options when required:

    | Option | Result |
    |---|---|
    | `--deps-only` | Install module dependencies without staging module inputs. |
    | `--force-deps` | Bypass the dependency cache and reinstall dependencies. |
    | `--skip <domain,...>` | Skip the modules identified by the listed internal domain names during initialization. |
    | `--skip-catalog` | Do not copy the catalog samples. |

    For example:

    ```bash title="Run on: OIM host"
    ./omnia.sh --setup-venv --skip telemetry,utils
    ```

4. Load the installed environment and activate the virtual environment:

    ```bash title="Run on: OIM host"
    source /opt/omnia/activate-omnia.sh
    ```

    If `OMNIA_DATA_PATH` is customized, source
    `<OMNIA_DATA_PATH>/activate-omnia.sh` instead.

## Verification

Verify the files and virtual-environment commands created by setup:

```bash title="Run on: OIM host"
test -f /etc/omnia/omnia.env
test -f /etc/profile.d/omnia-env.sh
test -f "$OMNIA_DATA_PATH/activate-omnia.sh"
test -x "$OMNIA_VENV_PATH/bin/python"
python --version
ansible --version
```

Python must report version 3.11 or later. Unless `--skip-catalog` was used,
confirm that the catalog directory contains the supplied samples:

```bash title="Run on: OIM host"
find "$OMNIA_DATA_PATH/catalog" -maxdepth 1 -type f
```

## Next steps

- Continue with the [Repository Manager flow](../repo_manager/index.md) to
  configure repository inputs, deploy Pulp, synchronize catalog content, and
  generate the repository output required by Image Build Manager.

## Troubleshooting

- **`SYSTEM_ADMIN_NIC_IPV4` is missing or invalid**: Set a valid IPv4 address
  in `src/main/omnia.env` and rerun setup.
- **The administrative address is not local**: Select an address assigned to
  an OIM network interface.
- **The hostname check fails**: Make `SYSTEM_HOSTNAME` match `hostname -s`. A
  domain-name mismatch is reported as a warning.
- **Python is unavailable or older than 3.11**: Install a supported Python
  version and rerun setup.
- **Setup was interrupted**: Rerun `./omnia.sh --setup-venv`; Main reports that
  the virtual environment might be incomplete.
- **A dependency remains cached**: Rerun setup with `--force-deps`.
