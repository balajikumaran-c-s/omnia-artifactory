# Deploy the Telemetry Stack

## Overview

The Telemetry entry point validates the three runtime input files, deploys the
required sinks, deploys each enabled source, reconciles the generated
Kustomize manifests, checks pod state, and writes `telemetry_status.yml`.
Running without tags performs validation followed by deployment. Cleanup,
precheck, upgrade, rollback, and connection-export workflows run only when
their tags are selected explicitly.

## Prerequisites

- Meet the platform requirements on the [Telemetry landing page](index.md).
- Export `OMNIA_DATA_PATH` and `OMNIA_PROJECT_NAME`, or accept `/opt/omnia` and
  `project_default`.
- Provide an Orchestrator YAML inventory whose `kube_vip_group` contains the
  service Kubernetes VIP. The inventory must also contain service control-plane
  and worker groups; LDMS additionally uses its Slurm groups.
- Ensure that the OIM can reach the VIP over SSH as `root` and that `kubectl`
  on the VIP can access the cluster.
- Make the shared `k8s_cluster_mount` available on the Kubernetes nodes. LDMS
  also requires the configured `slurm_cluster_mount` on the Slurm nodes.

## Procedure

1. From the Omnia source root, initialize the shared virtual environment and
   Telemetry runtime files:

    ```bash title="Run on: OIM"
    set -a
    source src/main/omnia.env
    set +a
    ./omnia.sh --setup-venv
    ```

    Telemetry templates are staged under
    `<OMNIA_DATA_PATH>/telemetry/input/<project>/`. Running
    `src/telemetry/domain-init.sh --force` overwrites existing project inputs;
    use it only when that is intended.

2. Edit all three project input files without removing their keys:

    - `telemetry_config.yml`: inventory, sources, bridges, sinks, and
      source-specific settings.
    - `telemetry_storage_config.yml`: replica counts and CPU, memory, and PVC
      settings.
    - `telemetry_packages.yml`: online/offline mode, repository URL, shared
      Kubernetes and Slurm mounts, registry, images, charts, repositories, and
      Python modules.

3. Run the opt-in environment precheck:

    ```bash title="Run on: OIM"
    ./omnia.sh -r telemetry --tags precheck
    ```

    It validates the VIP and SSH access, control-plane and worker readiness,
    non-Telemetry pod health, and the source-specific PowerScale and LDMS
    prerequisites when those sources are enabled.

4. Validate only the input contract when desired:

    ```bash title="Run on: OIM"
    ./omnia.sh -r telemetry --tags validate
    ```

5. Deploy the enabled configuration:

    ```bash title="Run on: OIM"
    ./omnia.sh -r telemetry --tags deploy
    ```

    The credential role creates an encrypted `telemetry_credentials.yml` and
    prompts only for empty credentials required by the enabled sources.

    The equivalent command from `src/telemetry` is:

    ```bash title="Run on: OIM"
    ansible-playbook playbooks/telemetry.yml --tags deploy
    ```

## Verification

1. Read the generated status file:

    ```bash title="Run on: OIM"
    cat /opt/omnia/telemetry/output/project_default/telemetry_status.yml
    ```

    Adjust the path when `OMNIA_DATA_PATH` or `OMNIA_PROJECT_NAME` differs.
    Confirm `overall_status: success` and check that each requested sink,
    source, and bridge is `deployed`. Disabled components are `skipped`.

2. On the Kubernetes VIP, inspect the namespace:

    ```bash title="Run on: Kubernetes VIP"
    kubectl get pods -n telemetry
    ```

    Deployment fails when a pod reaches `CrashLoopBackOff`, `Error`,
    `ImagePullBackOff`, `ErrImagePull`, `InvalidImageName`, or
    `CreateContainerConfigError`.

## Next steps

- Use the source-specific guides from the [Telemetry landing page](index.md) to
  configure and verify each data path.
- Export [Kafka](configure_external_kafka.md) or
  [Victoria](configure_external_victoria.md) connection details when external
  systems must publish or query Telemetry data.
- To remove all Telemetry runtime resources while preserving PVCs and Kafka
  identity metadata, run `./omnia.sh -r telemetry --tags cleanup`. Pass
  `-e Delete_volume=true` only when persistent volumes must also be deleted.

## Troubleshooting

- **Input validation fails:** Correct every file listed in the validation
  output. Validation applies JSON Schema checks and cross-field logic to all
  three input files.
- **`kube_vip` is missing:** Set `cluster_inventory` to a valid inventory with
  `all.children.kube_vip_group.hosts`.
- **The VIP is unreachable:** Restore root SSH connectivity from the OIM to the
  inventory VIP.
- **A pod is in an error state:** Run
  `kubectl logs -n telemetry <pod-name>` on the VIP and correct the reported
  image, configuration, or secret issue.
- **An LDMS node was skipped:** Review `deploy_unreachable_nodes.ldms` in the
  status file. Unreachable LDMS nodes are recorded without failing an otherwise
  successful deployment; failures returned by reachable nodes remain fatal.
