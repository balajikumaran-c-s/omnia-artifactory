# Build Stream Input/Output Contract

**Deployment module**: Build Stream | **CLI identifier**: `build_stream` | **Collection**: `omnia.build_stream`

## Input contract

The current Build Stream entry playbook resolves its runtime directories to:

```text
$OMNIA_DATA_PATH/build_stream/input/project_default/
$OMNIA_DATA_PATH/build_stream/output/project_default/
```

Set `OMNIA_DATA_PATH` as required. The executable
`build_stream_setup` role currently fixes the project directory to
`project_default`; do not select another `OMNIA_PROJECT_NAME` for this
workflow.

### `build_stream_config.yml`

`domain-init.sh` stages the consolidated configuration at:

```text
$OMNIA_DATA_PATH/build_stream/input/project_default/build_stream_config.yml
```

The schema is
`plugins/module_utils/input_validation/schema/build_stream_config.json`.

| Field | Requirement | Default | Purpose |
|---|---|---|---|
| `enable_build_stream` | Required | None | Enables or disables the Build Stream deployment. |
| `build_stream_host_ip` | Required when enabled | Empty | OIM address hosting the Build Stream Manager (BSM) API. |
| `build_stream_port` | Required when enabled | `8010` | BSM API port from 1 through 65535. |
| `gitlab_host` | Required for GitLab execution | Empty | Address of the target GitLab host reachable from the OIM. |
| `gitlab_project_name` | Optional | `omnia-catalog` | Project created and managed by the workflow. |
| `gitlab_project_visibility` | Optional | `private` | `private`, `internal`, or `public`. |
| `gitlab_default_branch` | Optional | `main` | Branch used by repository and API operations. |
| `gitlab_https_port` | Optional | `443` | GitLab HTTPS port. |
| `gitlab_min_storage_gb` | Optional | `20` | Minimum free storage checked before installation. |
| `gitlab_min_memory_gb` | Optional | `4` | Minimum memory checked before installation. |
| `gitlab_min_cpu_cores` | Optional | `2` | Minimum CPU count checked before installation. |
| `gitlab_puma_workers` | Optional | `2` | GitLab Puma worker count. |
| `gitlab_sidekiq_concurrency` | Optional | `10` | GitLab Sidekiq concurrency. |

Unknown configuration fields are rejected.

### Credentials

The credential workflow creates these root-owned, Ansible Vault-protected
files beside the configuration:

```text
build_stream_credentials.yml
.build_stream_credentials_key
```

The credential file contains the GitLab root and SSH passwords and, when Build
Stream is enabled, the BSM authentication and PostgreSQL credentials used by
the deployed services. The source credential schema and rules are under
`plugins/module_utils/input_validation/schema/`.

Build Stream infrastructure has no required upstream module status contract.
The GitLab pipelines subsequently upload catalog and module input files to BSM
jobs and invoke Repo Manager, Image Build Manager, Orchestrator, or Telemetry
as selected by the pipeline.

### GitLab project inputs

The managed project uses these source-controlled inputs:

| Input | Behavior |
|---|---|
| `catalog_rhel.json` | A change starts the build pipeline. |
| `input/orchestrator/pxe_mapping_file.csv` | A change starts the deploy pipeline. |
| `PIPELINE_TYPE` | API or trigger value selecting `build`, `deploy`, or `cleanup`. |

The PXE CSV follows the Orchestrator mapping contract:

```text
FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
```

## Output contract

### `build_stream_status.yml`

The `prepare` flow writes:

```text
$OMNIA_DATA_PATH/build_stream/output/project_default/build_stream_status.yml
```

| Field | Purpose |
|---|---|
| `overall_status` | The current writer records `prepared` after infrastructure preparation. |
| `build_stream_host` | Configured BSM host address. |
| `build_stream_port` | Configured BSM port. |
| `gitlab_host` | Configured GitLab host. |
| `gitlab_https_port` | Configured GitLab HTTPS port. |
| `gitlab_url` | Computed GitLab HTTPS base URL. |
| `bsm_api_url` | Computed BSM HTTPS base URL. |

The source does not generate `pipeline_status.yml` or
`catalog_manifest.yml` in the Build Stream project output directory.
Pipeline and job results are exposed through GitLab and BSM job state.

### Managed services and GitLab resources

| Output | Purpose |
|---|---|
| `omnia_postgres.service` | PostgreSQL state used by BSM. |
| `omnia_build_stream.service` | BSM FastAPI service. |
| `playbook-watcher.service` | Executes queued module playbooks. |
| GitLab project | Contains the catalog, module inputs, and parent/child CI pipeline files. |
| GitLab runner | Executes the managed build, deploy, and cleanup pipelines. |
| `miscellaneous/failed_nodes.json` | Deploy-pipeline retry state when node PXE boot or registration fails. |

A successful preparation is verified from `build_stream_status.yml`, the
three OIM services, the BSM health endpoint, and the managed GitLab project and
runner.

## Workflow tags

| Tag | Implemented behavior |
|---|---|
| No tag | Setup, validation, credentials, BSM preparation, and GitLab execution. |
| `precheck` | Checks the existing environment without collecting credentials. |
| `validate` | Validates `build_stream_config.yml`. |
| `credentials` | Collects or updates Build Stream credentials. |
| `prepare` | Deploys PostgreSQL, BSM, and the playbook watcher. |
| `execute` | Deploys and configures GitLab CI/CD. |
| `build` | Runs both `prepare` and `execute`. |
| `cleanup` | Removes the GitLab and Build Stream infrastructure. |
| `upgrade` | Runs the current placeholder upgrade flow. |
| `rollback` | Runs the current placeholder rollback flow. |

## Related documentation

- [Build Stream](../../HowTo/build_stream/index.md)
- [Deploy GitLab and Build Stream](../../HowTo/build_stream/deploy_gitlab.md)
- [Execute the build pipeline](../../HowTo/build_stream/execute_build_pipeline.md)
- [Execute the deploy pipeline](../../HowTo/build_stream/execute_deploy_pipeline.md)
- [Orchestrator contract](orchestrator_contract.md)
