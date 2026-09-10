# Verify VAST Telemetry

## Overview

Verify the Kubernetes integration objects that expose the configured VAST
Prometheus endpoint to the shared VictoriaMetrics scraper.

## Prerequisites

- Complete [Configure VAST Telemetry](configure_vast.md).
- Run Kubernetes commands on the configured VIP.

## Procedure

Inspect the VAST Service, Endpoints, and optional credential Secret:

```bash title="Run on: Kubernetes VIP"
kubectl get service vast-external -n telemetry
kubectl get endpoints vast-external -n telemetry
kubectl get secret vast-telemetry-credentials -n telemetry
```

The Secret is expected for basic authentication or CA-signed TLS; it may not be
created when authentication is `none` and TLS is self-signed.

## Verification

The endpoint address and metrics port must match `vast_configuration`. Confirm
`sources.vast.metrics: deployed` in `telemetry_status.yml`, then query the
generated VictoriaMetrics endpoint for an end-to-end ingestion check. If VAST
logs are enabled, confirm `sources.vast.logs: deployed`, verify
`vlagent.available: true` in the external Victoria output, and query for a log
sent by VAST.

The current status aggregation can mark enabled VAST channels as `deployed`
without using the VAST-specific component check. Treat the status as a summary
only and rely on the resource and sink queries above for verification.

## Next steps

- Export the [VictoriaMetrics query and UI endpoints](configure_external_victoria.md).
- Retain the VAST CA file on the OIM when CA-signed TLS is used.

## Troubleshooting

- **The Service is absent:** Confirm `telemetry_sources.vast.metrics_enabled` is
  `true` and rerun deployment.
- **The endpoint is wrong:** Correct `vast_endpoint`, `vast_metrics_port`, or
  `metrics_path` and redeploy.
- **Scraping fails:** Check endpoint reachability, credentials, CA content,
  scrape interval, and scrape timeout.
