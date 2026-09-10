# Operations & Maintenance


Day-2 operations for managing a running Omnia cluster. These guides cover
common administrative tasks you will perform after the initial deployment is
complete---scaling the cluster, re-provisioning nodes, upgrading to a new
Omnia version, rolling back a failed upgrade, managing logs, hardening
security, and cleaning up the OIM when a fresh start is needed.

!!! tip

    If you have not yet deployed your cluster, start with the
    [Get Started Tutorials](../GetStarted/index.md). The procedures in this section assume a
    working Omnia environment.

## Environment and content lifecycle

- [Maintain the Main environment](maintain_main_environment.md) to audit
  dependency declarations or remove the installed OIM environment.
- [Update repositories after catalog changes](repo_manager/updating_local_repositories.md)
  to synchronize revised catalog content and regenerate `repo_status.yml`.
- [Resynchronize local RPM repositories](repo_manager/local_repository_resync.md)
  to force selected or all catalog-required RPM remotes to check upstream.

## Node lifecycle

- [Add nodes](add_nodes.md) through the direct Orchestrator workflow and use a
  custom inventory to PXE boot only the new physical nodes.
- [Remove Slurm compute nodes](remove_slurm_nodes.md) omitted from the current
  desired mapping, with source-defined handling for active jobs.
- [Add nodes through Build Stream](build_stream/add_nodes.md) by committing the
  revised Orchestrator mapping and running the deploy pipeline.
- [Reprovision a cluster](reprovision_cluster.md) when existing nodes must be
  provisioned again.

## Build Stream lifecycle

- [Update the Build Stream catalog](build_stream/update_catalog.md).
- [Clean up image groups](build_stream/cleanup_operations.md).
- [Retry a failed pipeline](build_stream/retry_pipelines.md).

## Diagnostics and recovery

- [Collect cluster logs](collect_cluster_logs.md) with the Utils `collect`
  workflow.
- Review the [Slurm configuration roles](slurm_configuration_roles.md) before
  integrating the source roles into an administrator-maintained playbook.
- Use [Log Management](log_management.md) for general log inspection.


















