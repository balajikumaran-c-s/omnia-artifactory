# Verify UFM Telemetry

## Overview

Verify the Kubernetes integration objects that expose the configured UFM
Prometheus endpoint to the shared VictoriaMetrics scraper.

## Prerequisites

- Complete [Configure UFM Telemetry](configure_ufm.md).
- Run Kubernetes commands on the configured VIP.

## Procedure

Inspect the UFM Service, Endpoints, and optional credential Secret:

```bash title="Run on: Kubernetes VIP"
kubectl get service ufm-external -n telemetry
kubectl get endpoints ufm-external -n telemetry
kubectl get secret ufm-telemetry-credentials -n telemetry
```

The Secret is expected for basic authentication or CA-signed TLS; it may not be
created when authentication is `none` and TLS is self-signed.

## Verification

The endpoint address and metrics port must match `ufm_configuration`. Confirm
`sources.ufm.metrics: deployed` in `telemetry_status.yml`, then query the
generated VictoriaMetrics endpoint for an end-to-end ingestion check. If UFM
logs are enabled, confirm `sources.ufm.logs: deployed`, verify
`vlagent.available: true` in the external Victoria output, and query for a log
sent by UFM.

## Next steps

- Export the [VictoriaMetrics query and UI endpoints](configure_external_victoria.md).
- Retain the UFM CA file on the OIM when CA-signed TLS is used.

## Troubleshooting

- **The Service is absent:** Confirm `telemetry_sources.ufm.metrics_enabled` is
  `true` and rerun deployment.
- **The endpoint is wrong:** Correct `ufm_endpoint` or `ufm_metrics_port` and
  redeploy.
- **Scraping fails:** Check endpoint reachability, credentials, CA content,
  scrape interval, and scrape timeout.
