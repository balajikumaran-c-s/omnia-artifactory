# Verify LDMS Telemetry

## Overview

Verify the LDMS aggregator, store, Kafka topic, sampler preparation, and the
optional Vector-LDMS bridge.

## Prerequisites

- Complete [Configure LDMS Telemetry](configure_ldms.md).
- Run Kubernetes commands on the configured VIP.

## Procedure

Run the resource checks used by the LDMS role:

```bash title="Run on: Kubernetes VIP"
kubectl get statefulset nersc-ldms-aggr -n telemetry \
  -o jsonpath='{.status.readyReplicas}'
kubectl get pods -l app=nersc-ldms-store -n telemetry \
  -o jsonpath='{.items[0].status.phase}'
kubectl get kafkatopic ldms -n telemetry
kubectl get deployment vector-ldms -n telemetry \
  -o jsonpath='{.status.readyReplicas}'
```

The Vector deployment is expected only when the bridge is enabled.

## Verification

The aggregator must have a ready replica, the store must be `Running`, and the
`ldms` topic must exist. If Vector-LDMS is enabled, its deployment must have a
ready replica. Confirm `sources.ldms.metrics: deployed` and the expected bridge
state in `telemetry_status.yml`.

The deployment status does not prove ingestion. Read records from the `ldms`
topic and, when Vector-LDMS is enabled, query VictoriaMetrics for an LDMS
metric to complete end-to-end verification.

## Next steps

- Use [Verify Vector-LDMS](verify_vector_ldms.md) for the bridge-specific check.
- Review `deploy_unreachable_nodes.ldms` in the status file and restore omitted
  nodes before rerunning deployment.

## Troubleshooting

- **The aggregator is not ready:** Confirm the selected reachable Slurm control
  node has a readable Munge key and that the Kubernetes deployment is healthy.
- **The store is not Running:** Inspect the `nersc-ldms-store` pod logs.
- **A sampler is unavailable:** Check `ldmsd` and the configured sampler port on
  that Slurm node.
