# Repository Manager Input/Output Contract

**Deployment module**: Repository Manager | **CLI identifier**: `repo_manager` | **Collection**: `omnia.repo_manager`

## Input contract

### Environment

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `SYSTEM_ADMIN_NIC_IPV4` | Yes | None | OIM admin-network IPv4 used by Pulp. |
| `CATALOG_FILE_PATH` | Yes | None | Absolute path to an existing catalog file with a `.json` extension. |
| `OMNIA_DATA_PATH` | No | `/opt/omnia` | Root Omnia runtime data directory. |
| `REPO_MANAGER_DATA_PATH` | No | `<OMNIA_DATA_PATH>/repo_manager` | Repo Manager runtime root. |
| `OMNIA_PROJECT_NAME` | No | `project_default` | Active input and output project. |

### Files

`domain-init.sh` stages the two source YAML inputs under
`<REPO_MANAGER_DATA_PATH>/input/<project>/`. The catalog remains at the exact
path specified by `CATALOG_FILE_PATH`.

| Input | Required | Purpose |
|---|---|---|
| `repo_manager_config.yml` | Yes | Defines RPM repositories, container registries, and synchronization policies. |
| `repo_manager_endpoint_config.yml` | Yes | Defines the host-facing Pulp HTTPS endpoint. |
| Catalog JSON | Yes | Selects functional layers, groups, packages, OS versions, architectures, and sources to synchronize. |
| `repo_manager_config_credentials.yml` | After `prepare` | Ansible Vault-protected Pulp and registry credentials. |
| `.repo_manager_config_credentials_key` | After `prepare` | Key for the encrypted credential file. |

The generated credential files are root-owned and use mode `0600`.

### `repo_manager_config.yml`

Schema:
`plugins/module_utils/input_validation/schema/repo_manager_config.json`

| Field | Type | Required | Default | Purpose |
|---|---|---|---|---|
| `catalog_config` | object | No | None | Compatibility catalog reference; runtime selection uses `CATALOG_FILE_PATH`. |
| `repo_config` | string | Yes | None | Global RPM policy: `always` or `partial`. |
| `caching_policy` | boolean | No | `true` | Global RPM caching behavior. |
| `registries` | object or null | No | `null` | Custom container registries keyed by catalog registry name. |
| `repositories` | object | Yes | None | Repository definitions organized by OS version and architecture. |

Repository definitions are resolved independently for `x86_64` and `aarch64`.
Each catalog RPM source is matched by OS version, architecture, and
`reponame`. A referenced repository requires an explicit URL unless it is
`baseos`, `appstream`, or `codeready-builder` and usable RHEL subscription
content is available.

Repository entries support `url`, `gpgkey`, `policy`, `caching`, `priority`,
`sslcacert`, `sslclientkey`, and `sslclientcert`. A repository priority must be
from 1 through 100. Unknown configuration fields are rejected.

Configured private registries use `base_url`, `port`, `auth`, and `tls`. Basic
authentication refers to a credential through `auth.credentials.vault_path`;
credentials do not belong in the catalog or main configuration.

### `repo_manager_endpoint_config.yml`

Schema:
`plugins/module_utils/input_validation/schema/repo_manager_endpoint_config.json`

| Field | Type | Required | Default | Purpose |
|---|---|---|---|---|
| `pulp_server_port` | integer | Yes | `2225` | Host HTTPS port from 1 through 65535. |
| `pulp_server_ip` | IPv4 string | No | `SYSTEM_ADMIN_NIC_IPV4` | Host IP advertised to consumers. |

HTTPS is mandatory. Certificate paths are derived from
`REPO_MANAGER_DATA_PATH` and are not endpoint inputs.

### Catalog JSON

The catalog must contain `name`, `version`, `identifier`, `description`,
`functionallayer`, `groups`, and `packages` under its `catalog` object. Repo
Manager consumes these package fields:

| Field | Purpose |
|---|---|
| `name` | Upstream package, image, or artifact name. |
| `packagetype` | Selects the Repo Manager processing path. |
| `version` or `tag` | Package version or OCI image tag. |
| `sources[].architecture` | Selects `x86_64` or `aarch64`. |
| `sources[].version` | Selects one or more OS versions. |
| `sources[].reponame` | Maps RPM content to `repositories`. |
| `sources[].registry` | Maps an OCI image to a configured registry key. |
| `url` or `sources[].url` | Supplies the HTTP(S) URL for a direct artifact. |

Only packages reachable through selected functional layers and groups are
processed. Every referenced repository and non-public registry must resolve
before synchronization starts.

## Output contract

### `repo_status.yml`

**Location**:
`<REPO_MANAGER_DATA_PATH>/output/<project>/repo_status.yml`

**Producer**: `generate_local_repo_access` module, run by the `status` tag.

**Consumers**: Image Build Manager and cluster provisioning workflows.

| Field | Type | Purpose |
|---|---|---|
| `overall_status` | string | Aggregate readiness across selected catalog contexts. |
| `cluster_os_type` | string | Catalog operating-system type. |
| `execution_contexts` | list | Ordered OS-version and architecture contexts. |
| `overall_status_by_version` | object | Per-version `pending`, `success`, or `failed` state. |
| `repo_config` | string | Effective repository policy reported by the generator. |
| `repo_manager.port` | integer | Pulp HTTPS host port. |
| `repo_manager.certificates` | object | Public Pulp certificate path and directory. |
| `repositories.<version>.<architecture>` | object | RPM distribution URLs and optional DNF priorities. |
| `registries` | object | Configured private-registry endpoints and non-secret TLS settings. |
| `file_repos.<architecture>` | object | File and Python distribution URLs by content type and artifact. |
| `*_base_url` and `offline_*_path` | string | Content-type base URLs and backward-compatible URLs. |

Repository URLs come from actual Pulp distributions. A missing required
distribution makes the affected version and aggregate status `failed`. During
a multi-version download, `overall_status` remains `in_progress` while later
contexts are pending and becomes `success` only after every selected context
completes.

Registry authentication references, usernames, passwords, and tokens are not
written to `repo_status.yml`. Selective cleanup removes the stale status file;
the `status` tag regenerates it from the current Pulp state.

### Managed services and content

The `prepare` operation creates or configures these resources:

| Output | Location or name | Purpose |
|---|---|---|
| Systemd service | `pulp.service` | Enabled Pulp Podman Quadlet service. |
| Quadlet | `/etc/containers/systemd/pulp.container` | Pulp container definition. |
| HTTPS endpoint | `https://<pulp_server_ip>:<pulp_server_port>` | Pulp API, content, and OCI registry endpoint. |
| CA certificate | `<REPO_MANAGER_DATA_PATH>/pulp_config/settings/certs/pulp_webserver.crt` | Client trust certificate. |
| Pulp CLI | `/usr/local/bin/pulp` | Managed CLI configured for HTTPS access. |
| Host trust anchor | `/etc/pki/ca-trust/source/anchors/omnia-pulp.crt` | System CA trust. |

Content stored in Pulp is the authoritative downloadable output. Files under
`offline_repo` are staging content rather than the downstream contract.

### Runtime state and logs

| Output | Location | Purpose |
|---|---|---|
| Package status | `<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/<architecture>/<group>/status.csv` | Per-package and artifact state for a catalog group. |
| Group status | `<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/<architecture>/groups_status.csv` | Overall state for resolved groups. |
| Mirror indexes | `<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/mirror_status/` | Catalog package ownership and Pulp mirror state. |
| Execution summary | `<REPO_MANAGER_DATA_PATH>/log/<os>/catalog_execution_summary.yml` | Ordered contexts and aggregate run state. |
| Top-level Ansible log | `/var/log/omnia/repo_manager/repo_manager.log` | Repo Manager playbook log. |

Runtime status and mirror files support idempotent reruns. Downstream components
consume `repo_status.yml`.

## Related documentation

- [Repository Manager](../../HowTo/repo_manager/index.md)
- [Create Local Repositories](../../HowTo/repo_manager/configure_repos.md)
