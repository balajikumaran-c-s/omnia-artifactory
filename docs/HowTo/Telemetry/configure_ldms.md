# Configure LDMS Telemetry

## Overview

LDMS collects compute, control, login, and login-compiler node metrics. Omnia
configures `ldmsd` samplers on reachable Slurm nodes, deploys the LDMS
aggregator and store in Kubernetes, and publishes to the Kafka `ldms` topic.
When enabled, Vector-LDMS consumes that topic and forwards metrics through
`vmagent-vector` to VictoriaMetrics.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Ensure `cluster_inventory` contains at least one Slurm control node and one
  Slurm compute node. The precheck requires `slurmctld` on control nodes and
  `slurmd` on compute nodes.
- Ensure at least one reachable Slurm control node has a readable
  `/etc/munge/munge.key`.
- Make the shared Slurm and Kubernetes mounts configured by
  `telemetry_packages.yml` available to their respective nodes.
- Provide `ldms_sampler_password` when prompted.

## Procedure

1. Enable the LDMS source in `telemetry_config.yml`. Kafka is its only
   supported collection target:

    ```yaml
    telemetry_sources:
      ldms:
        metrics_enabled: true
        collection_targets:
          - kafka

    telemetry_bridges:
      vector_ldms:
        metrics_enabled: true
    ```

    Disable `vector_ldms.metrics_enabled` only when LDMS data should remain in
    Kafka and should not be forwarded to VictoriaMetrics.

2. Configure ports and sampler plugins. Aggregator and store ports accept
   `6001` through `6100`; the sampler port accepts `10001` through `10100`.

    ```yaml
    ldms_configurations:
      agg_port: 6001
      store_port: 6001
      sampler_port: 10001
      sampler_plugins:
        - plugin_name: meminfo
          config_parameters: ""
          activation_parameters: "interval=30000000"
    ```

    Supported plugin names are `meminfo`, `procstat2`, `vmstat`, `loadavg`,
    `slurm_sampler`, and `procnetdev2`.

3. Keep the LDMS and Vector image entries in `telemetry_packages.yml` and their
   resource sections in `telemetry_storage_config.yml` consistent with the
   images and capacity available to the cluster.

4. Run the precheck and deployment:

    ```bash title="Run on: OIM"
    ./omnia.sh -r telemetry --tags precheck
    ./omnia.sh -r telemetry --tags deploy
    ```

## Verification

On the Kubernetes VIP, repeat the checks used by the LDMS role:

```bash title="Run on: Kubernetes VIP"
kubectl get statefulset nersc-ldms-aggr -n telemetry
kubectl get pods -l app=nersc-ldms-store -n telemetry
kubectl get kafkatopic ldms -n telemetry
kubectl get deployment vector-ldms -n telemetry
```

The Vector deployment is expected only when the bridge is enabled. Confirm
`sources.ldms.metrics: deployed` and the expected `bridges.vector_ldms` value
in `telemetry_status.yml`. Review `deploy_unreachable_nodes.ldms` for skipped
Slurm nodes.

## Next steps

- Use [Verify LDMS Telemetry](verify_ldms.md) for the standalone checks.
- Use [Verify Vector-LDMS](verify_vector_ldms.md) when the VictoriaMetrics route
  is enabled.

## Troubleshooting

- **Precheck reports missing Slurm groups or services:** Correct the configured
  inventory and start `slurmctld` and `slurmd` on the required nodes.
- **Aggregator preparation fails:** Ensure at least one reachable control node
  has a readable Munge key.
- **A sampler does not start:** Confirm that its generated configuration is
  present, the configured sampler port is available, and `ldmsd` can start.
  Omnia opens that port automatically when `firewalld` is active.
- **Vector-LDMS is rejected:** The bridge requires the LDMS source to be
  enabled. It also requires Kafka and VictoriaMetrics support.

