# Configure PowerScale Telemetry

## Overview

PowerScale metrics use the existing PowerScale CSI deployment, CSM Metrics
PowerScale, an OTEL Collector, the shared `vmagent`, and VictoriaMetrics.
PowerScale logs use the shared VLAgent syslog endpoint and VictoriaLogs. Omnia
installs or upgrades the `karavi-observability` Helm release; it does not deploy
or configure the PowerScale system itself.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Deploy the PowerScale CSI driver in the `isilon` namespace and ensure its pods
  are running. The `external-health-monitor-controller` is required for the CSI
  volume exporter health metrics.
- Provide a CSM Observability values YAML containing
  `karaviMetricsPowerscale.image`, `otelCollector.image`, and
  `cert-manager.enabled: true`. Do not enable PowerFlex, PowerStore, or PowerMax
  metrics in this file.
- Keep `images.powerscale.csm_metrics` and
  `images.powerscale.otel_collector` in `telemetry_packages.yml`; in offline
  mode, values-file image versions must match the package manifest.
- Provide `csi_username` and `csi_password` when prompted.

## Procedure

1. Configure the PowerScale source and its required metrics target:

    ```yaml
    telemetry_sources:
      powerscale:
        metrics_enabled: true
        logs_enabled: false
        collection_targets:
          - victoria_metrics

    powerscale_configurations:
      otel_collector_storage_size: "5Gi"
      csm_observability_values_file_path: "/path/to/values.yaml"
    ```

2. To prepare PowerScale log ingestion as well, keep metrics enabled, set
   `logs_enabled: true`, and add `victoria_logs` to `collection_targets`.
   The root workflow imports the PowerScale source only when
   `metrics_enabled: true`; a logs-only configuration is not supported.

3. Keep the `csm_metrics_powerscale_storage` and
   `csi_volume_exporter_storage` sections in `telemetry_storage_config.yml`.

4. Run the precheck and deployment:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags precheck
    ./omnia.sh --run telemetry --tags deploy
    ```

5. When logs are enabled, export the generated VLAgent target and PowerScale
   `isi audit` commands:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags external_victoria
    ```

    Run the commands recorded under `powerscale.isi_audit_commands` in the
    generated connection-details file on the PowerScale system.

## Verification

On the Kubernetes VIP, verify the same resources inspected by deployment:

```bash title="Run on: Kubernetes VIP"
kubectl get pods -n telemetry -l app.kubernetes.io/name=karavi-metrics-powerscale
kubectl get deployment otel-collector -n telemetry
```

Confirm `sources.powerscale.metrics: deployed` in `telemetry_status.yml`. When
logs are enabled, confirm `sources.powerscale.logs: deployed` and check that the
generated external Victoria file reports `vlagent.available: true`.

The log status confirms that the shared VLAgent is available; it does not
configure PowerScale log forwarding or prove ingestion. Run the exported
`isi audit` commands and query VictoriaLogs for an end-to-end check. Likewise,
query VictoriaMetrics to confirm that PowerScale metrics are being ingested.

## Next steps

- Use [Verify PowerScale Telemetry](verify_powerscale.md) for the repeatable
  resource checks.
- Use [Export Victoria Connection Details](configure_external_victoria.md) to
  obtain write, query, and syslog endpoints.

## Troubleshooting

- **Validation cannot find the values file:** Set an existing path in
  `csm_observability_values_file_path` and ensure it contains valid YAML.
- **The values file is rejected:** Enable cert-manager, provide the required
  PowerScale and OTEL images, disable unsupported storage metrics, and align
  image versions with `telemetry_packages.yml` in offline mode.
- **PowerScale precheck warns about privileges:** Verify the PowerScale account
  has the permissions required by the enabled metrics or log path.
- **CSI volume exporter is skipped:** Enable the external health monitor in the
  PowerScale CSI driver and rerun Telemetry.
