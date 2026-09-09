# Verify the Vector-LDMS Pipeline

## Overview

Vector-LDMS consumes the Kafka `ldms` topic, transforms LDMS records, and sends
metrics through `vmagent-vector` to the VictoriaMetrics vminsert endpoint.

## Prerequisites

- Complete [Configure LDMS Telemetry](configure_ldms.md) with
  `telemetry_bridges.vector_ldms.metrics_enabled: true`.
- Ensure both Kafka and VictoriaMetrics are deployed.

## Procedure

On the Kubernetes VIP, inspect the bridge, its write buffer, and the topic:

```bash title="Run on: Kubernetes VIP"
kubectl get deployment vector-ldms vmagent-vector -n telemetry
kubectl get pods -n telemetry -l app=vector-ldms
kubectl get kafkatopic ldms -n telemetry
```

The deployment workflow waits for the Vector-LDMS rollout and for its pods to
be Ready.

## Verification

Both deployments must have ready replicas, the `ldms` topic must exist, and
`bridges.vector_ldms: deployed` must appear in `telemetry_status.yml`.

These checks establish bridge readiness. Query VictoriaMetrics for an LDMS
metric to verify that records traverse the complete pipeline.

## Next steps

- Export [VictoriaMetrics connection details](configure_external_victoria.md)
  to obtain the query and UI endpoints.
- Review [LDMS verification](verify_ldms.md) when the upstream aggregator,
  store, or samplers are not healthy.

## Troubleshooting

- **Vector-LDMS is skipped:** Enable both the LDMS source and its Vector bridge.
- **The bridge is not ready:** Inspect the Vector-LDMS logs and confirm that
  Kafka, the `ldms` topic, and `vmagent-vector` are available.
- **The bridge runs but no data arrives:** Verify the upstream LDMS aggregator,
  store, and sampler configuration first.
