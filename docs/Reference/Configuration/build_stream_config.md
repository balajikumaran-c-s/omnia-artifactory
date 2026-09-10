# build_stream_config.yml

The consolidated BuildStreaM configuration controls the BuildStreaM Manager
(BSM) API and the managed GitLab deployment. It replaces the former separate
BuildStreaM and GitLab configuration files.

## Location

```text
$OMNIA_DATA_PATH/build_stream/input/project_default/build_stream_config.yml
```

The current entry playbook fixes the project directory to `project_default`.

## Configuration parameters

| Parameter | Type | Requirement | Default |
|---|---|---|---|
| `enable_build_stream` | boolean | Required | None |
| `build_stream_host_ip` | IPv4 string | Required when enabled | Empty |
| `build_stream_port` | integer | Required when enabled | `8010` |
| `gitlab_host` | IPv4 string | Required for GitLab execution | Empty |
| `gitlab_project_name` | string | Optional | `omnia-catalog` |
| `gitlab_project_visibility` | string | Optional: `private`, `internal`, or `public` | `private` |
| `gitlab_default_branch` | string | Optional | `main` |
| `gitlab_https_port` | integer | Optional | `443` |
| `gitlab_min_storage_gb` | integer | Optional | `20` |
| `gitlab_min_memory_gb` | integer | Optional | `4` |
| `gitlab_min_cpu_cores` | integer | Optional | `2` |
| `gitlab_puma_workers` | integer | Optional | `2` |
| `gitlab_sidekiq_concurrency` | integer | Optional | `10` |

Ports must be from 1 through 65535. Resource minimums and worker counts must be
positive integers. Unknown fields are rejected.

`PIPELINE_TYPE` is a GitLab pipeline variable, not a field in this YAML file.

## Usage example

```yaml title="File: /opt/omnia/build_stream/input/project_default/build_stream_config.yml"
enable_build_stream: true
build_stream_host_ip: "172.16.107.254"
build_stream_port: 8010
gitlab_host: "172.16.107.20"
gitlab_project_name: "omnia-catalog"
gitlab_project_visibility: "private"
gitlab_default_branch: "main"
gitlab_https_port: 443
gitlab_min_storage_gb: 20
gitlab_min_memory_gb: 4
gitlab_min_cpu_cores: 2
gitlab_puma_workers: 2
gitlab_sidekiq_concurrency: 10
```

## Related configuration

- [Main environment](omnia_env.md)
- [BuildStreaM contract](../domain_contracts/build_stream_contract.md)
- [Deploy GitLab and BuildStreaM](../../HowTo/build_stream/deploy_gitlab.md)

