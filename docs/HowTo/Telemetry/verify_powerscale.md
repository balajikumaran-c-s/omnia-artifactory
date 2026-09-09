# Verify PowerScale Telemetry

## Overview

Verify the PowerScale CSM Metrics workload, OTEL Collector, and the deployment
status recorded by Telemetry.

## Prerequisites

- Complete [Configure PowerScale Telemetry](configure_powerscale.md).
- Run Kubernetes commands on the configured VIP.

## Procedure

Inspect the resources checked or created by the PowerScale workflow:

```bash title="Run on: Kubernetes VIP"
kubectl get pods -n telemetry \
  -l app.kubernetes.io/name=karavi-metrics-powerscale
kubectl get deployment otel-collector -n telemetry
kubectl get pvc otel-collector-data -n telemetry
kubectl get service otel-collector -n telemetry
```

When logs are enabled, inspect the shared VLAgent:

```bash title="Run on: Kubernetes VIP"
kubectl get pods -n telemetry -l app.kubernetes.io/name=vlagent
```

## Verification

The PowerScale metrics pod and OTEL Collector must be Running. Confirm
`sources.powerscale.metrics: deployed` in `telemetry_status.yml`; when logs are
enabled, confirm `sources.powerscale.logs: deployed` and a generated VLAgent
endpoint.

The log status confirms VLAgent availability; it does not prove that the
PowerScale system is forwarding logs. Query VictoriaMetrics for a PowerScale
metric and VictoriaLogs for a forwarded audit record to complete end-to-end
verification.

## Next steps

- Use [Export Victoria Connection Details](configure_external_victoria.md) to
  obtain query and syslog endpoints.
- Run the generated PowerScale `isi audit` commands when log forwarding is
  required.

## Troubleshooting

- **CSM Metrics is absent:** Confirm the PowerScale CSI deployment and the
  `karavi-observability` Helm release.
- **OTEL Collector is not ready:** Inspect its pod logs and PVC, then check the
  values file used by the release.
- **Health metrics are absent:** Ensure the CSI driver's external health monitor
  container is running.
