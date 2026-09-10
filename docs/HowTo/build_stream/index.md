# BuildStreaM

## Overview

BuildStreaM provides the GitLab and BuildStreaM Manager (BSM) services used
for catalog-driven image-build, deployment, and cleanup pipelines. BSM runs as
a FastAPI service on the Omnia Infrastructure Manager (OIM), and the
playbook-watcher executes the requested Omnia module playbooks.

The managed GitLab project routes:

```text
catalog_rhel.json change
        -> build pipeline -> Repo Manager -> Image Build Manager

input/orchestrator/pxe_mapping_file.csv change
        -> deploy pipeline -> Orchestrator

PIPELINE_TYPE=cleanup
        -> cleanup pipeline
```

## Prerequisites

- Complete the [OIM setup](../main/setup_oim.md).
- Use `project_default`. The current BuildStreaM entry playbook fixes its
  input and output directories to that project.
- Configure
  `/opt/omnia/build_stream/input/project_default/build_stream_config.yml`
  with `enable_build_stream: true`, the BSM host and port, and the GitLab
  host.
- Ensure the OIM can reach the GitLab host over SSH and HTTPS.
- Ensure the configured GitLab host satisfies the CPU, memory, and free-storage
  minimums in `build_stream_config.yml`.
- Ensure Podman and systemd are available on the OIM.
- Review the
  [BuildStreaM Domain Contract](../../Reference/domain_contracts/build_stream_contract.md)
  before deployment.

## Procedure

Choose the operation that matches the required workflow.

| Task | Use it to |
|---|---|
| [Deploy GitLab and BuildStreaM](deploy_gitlab.md) | Initialize inputs and credentials, deploy PostgreSQL, BSM, the watcher, GitLab, and the managed project and runner. |
| [Execute the build pipeline](execute_build_pipeline.md) | Commit `catalog_rhel.json` or select `PIPELINE_TYPE=build` to synchronize content and build images. |
| [Execute the deploy pipeline](execute_deploy_pipeline.md) | Commit `input/orchestrator/pxe_mapping_file.csv` or select `PIPELINE_TYPE=deploy` to provision mapped nodes. |
| [Add nodes](../../Operations/build_stream/add_nodes.md) | Add rows to the Orchestrator mapping contract and run the deploy pipeline. |
| [Initialize Telemetry](initialize_telemetry.md) | Configure and run Telemetry after a service Kubernetes cluster is available. |
| [Update the catalog](../../Operations/build_stream/update_catalog.md) | Modify the GitLab project’s root `catalog_rhel.json` and start a build. |
| [Clean up pipeline resources](../../Operations/build_stream/cleanup_operations.md) | Run the GitLab cleanup pipeline for selected image resources. |
| [Retry a pipeline](../../Operations/build_stream/retry_pipelines.md) | Retry a failed parent or downstream pipeline after correcting its cause. |

The BuildStreaM entry playbook supports no tag, `precheck`, `validate`,
`credentials`, `prepare`, `execute`, `build`, `cleanup`, `upgrade`,
and `rollback`. See the contract for the implemented behavior of each tag.

## Verification

After deploying BuildStreaM, inspect:

```bash title="Run on: OIM host"
cat /opt/omnia/build_stream/output/project_default/build_stream_status.yml
systemctl is-active omnia_postgres.service
systemctl is-active omnia_build_stream.service
systemctl is-active playbook-watcher.service
```

Confirm that `overall_status` is `prepared`, the reported BSM and GitLab
addresses match `build_stream_config.yml`, and all three services are active.
Then verify that the managed GitLab project and its runner are available at the
reported `gitlab_url`.

Pipeline results are reported through the GitLab pipeline and BSM job state.
BuildStreaM does not write `pipeline_status.yml` or
`catalog_manifest.yml` to its project output directory.

## Next steps

- Run the [build pipeline](execute_build_pipeline.md).
- After images are available, prepare
  `input/orchestrator/pxe_mapping_file.csv` and run the
  [deploy pipeline](execute_deploy_pipeline.md).
- Deploy [Telemetry](../Telemetry/deploy_telemetry.md) after the service
  Kubernetes cluster is reachable.

## Troubleshooting

- **The project input directory is not found:** Use
  `/opt/omnia/build_stream/input/project_default/`; the current entry
  playbook does not select another project.
- **Preparation fails:** Check
  `/var/log/omnia/build_stream/` and the status of PostgreSQL, BSM, and the
  watcher.
- **GitLab execution fails:** Confirm the GitLab host is reachable and meets
  the configured resource minimums.
- **A file commit does not start the expected pipeline:** Use
  `catalog_rhel.json` for builds and
  `input/orchestrator/pxe_mapping_file.csv` for deployments.
- **A module stage fails:** Follow its BSM job log path and inspect the
  corresponding module status contract.
