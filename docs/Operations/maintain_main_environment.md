# Maintain the Main environment

## Overview

Main provides one command to audit dependency-version declarations across the
modules and two cleanup modes for removing the installed environment. Normal
cleanup preserves runtime input, output, and log data. Full cleanup removes the
entire configured `OMNIA_DATA_PATH`.

## Prerequisites

- Run the commands from `src/main` in an Omnia source checkout.
- Use an account that can remove the installed system environment, virtual
  environment, and dependency cache.
- Before full cleanup, preserve any required content from `OMNIA_DATA_PATH`.
- Confirm the configured `OMNIA_DATA_PATH` and `OMNIA_VENV_PATH` before
  accepting a cleanup prompt.

## Procedure

1. Audit Python and Ansible Galaxy version declarations across the known Main
   modules:

    ```bash title="Run on: OIM host"
    cd src/main
    source /etc/profile.d/omnia-env.sh
    ./omnia.sh --check-deps
    ```

    The command scans each module's `requirements.txt` and `requirements.yml`.
    It exits with a nonzero status when it finds different version
    specifications for the same dependency.

2. To remove the virtual environment and installed Main environment while
   preserving runtime data, run:

    ```bash title="Run on: OIM host"
    ./omnia.sh --cleanup
    ```

    The command also removes the activation script, dependency cache, and the
    command helper files installed by Main. Enter exactly `yes` at the prompt
    to continue. Input, output, and logs under `OMNIA_DATA_PATH` are preserved.

3. To remove the installed environment and all content under
   `OMNIA_DATA_PATH`, run:

    ```bash title="Run on: OIM host"
    ./omnia.sh --cleanup --all
    ```

    Enter exactly `yes` at the prompt to confirm the full reset.

## Verification

After normal cleanup, verify that the installed environment is absent and the
runtime data path remains:

```bash title="Run on: OIM host"
test ! -e "$OMNIA_VENV_PATH"
test ! -e /etc/omnia/omnia.env
test ! -e /etc/profile.d/omnia-env.sh
test ! -e "$OMNIA_DATA_PATH/activate-omnia.sh"
test ! -e "$OMNIA_DATA_PATH/.data/deps-cache"
test -d "$OMNIA_DATA_PATH"
```

After `--cleanup --all`, verify that the configured data path is absent:

```bash title="Run on: OIM host"
test ! -e "$OMNIA_DATA_PATH"
```

## Next steps

- Align dependency specifications reported by `--check-deps` before creating
  the shared environment.
- After cleanup, [set up the OIM](../HowTo/main/setup_oim.md) again when a new environment is
  required.

## Troubleshooting

- **The dependency audit exits nonzero**: Review each reported module and
  version specification, then align the affected requirement files.
- **Cleanup is cancelled**: The prompt accepts only the exact response `yes`.
- **Files remain after cleanup**: Confirm that the account can remove the
  displayed system, virtual-environment, and data paths.
- **Runtime data was preserved unexpectedly**: Normal `--cleanup` is designed
  to preserve it. Use `--cleanup --all` only when the entire configured data
  path can be deleted.
