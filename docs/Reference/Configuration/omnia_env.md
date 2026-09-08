# omnia.env

`omnia.env` is the shared environment configuration used by `omnia.sh`, module
initialization scripts, and Ansible playbooks. Edit the source file before OIM
setup. The setup flow installs the resulting environment under `/etc/omnia`.

## Location

```text
Source:    src/main/omnia.env
Installed: /etc/omnia/omnia.env
```

Source the environment before running Omnia commands directly:

```bash
set -a
source src/main/omnia.env
set +a
```

## Variables

| Variable | Requirement | Source value | Purpose |
|---|---|---|---|
| `SYSTEM_ADMIN_NIC_IPV4` | Required | `172.16.107.254` | OIM admin-network IPv4 address used by platform services. |
| `OMNIA_DATA_PATH` | Optional | `/opt/omnia` | Root for persistent module data. |
| `OMNIA_PROJECT_NAME` | Optional | `project_default` | Selects each module's input and output project directory. |
| `SYSTEM_HOSTNAME` | Optional | `oim` | Short hostname of the OIM host. |
| `SYSTEM_DOMAIN_NAME` | Optional | `omnia.cluster` | Domain name of the OIM host. |
| `OMNIA_VENV_PATH` | Optional | `/opt/omnia/venv` | Shared Python virtual environment created during setup. |
| `OMNIA_VERSION` | Optional | `2.3` | Omnia release version. |
| `CATALOG_FILE_PATH` | Optional | `${OMNIA_DATA_PATH}/catalog/catalog_rhel.json` | Shared catalog path consumed by catalog-aware modules. |

The source also provides optional component path overrides:

```bash
# IMAGE_BUILD_MANAGER_DATA_PATH=${OMNIA_DATA_PATH}/image_build_manager
# REPO_MANAGER_DATA_PATH=${OMNIA_DATA_PATH}/repo_manager
# DISCOVERY_DATA_PATH=${OMNIA_DATA_PATH}/discovery
# ORCHESTRATOR_DATA_PATH=${OMNIA_DATA_PATH}/orchestrator
# TELEMETRY_DATA_PATH=${OMNIA_DATA_PATH}/telemetry
# BUILD_STREAM_DATA_PATH=${OMNIA_DATA_PATH}/build_stream
```

Do not add credentials to `omnia.env`; module credential playbooks create their
own encrypted credential files.

## Related configuration

- [Repo Manager configuration](repo_manager_config.md)
- [Image Build configuration](image_build_manager_config.md)
- [Orchestrator configuration](orchestrator_config.md)
