# Connect SmartFabric Manager to VictoriaMetrics

## Overview

The Telemetry external Victoria utility generates the connection values needed
to configure SmartFabric Manager (SFM) Prometheus remote write. SFM is an
external producer; the Telemetry module does not deploy, configure, or clean up
SFM. There is no SFM source role in the Telemetry domain.

## Prerequisites

- Deploy Telemetry with VictoriaMetrics enabled and Running in the `telemetry`
  namespace.
- Ensure the `vminsert-victoria-cluster` and
  `vmselect-victoria-cluster` LoadBalancer services have external IPs.
- Ensure the OIM can reach the Kubernetes VIP through root SSH.
- Have administrative access to SFM and its Prometheus pod.

## Procedure

1. Export the Victoria connection details:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags external_victoria
    ```

2. Read the generated SFM values from:

    ```text
    /opt/omnia/telemetry/output/project_default/external_victoria/external_victoria_connect_details.yml
    ```

    Adjust the path when the data root or project differs.

3. In SFM, open `Observability -> Settings -> Prometheus Remote Write` and use
   the generated values:

    - Enable remote write: `ON`
    - Message version: `v1`
    - Target name: `victoria`
    - Target URL: the generated
      `victoria_metrics.notes.sfm.vminsert_write_url`
    - TLS server certificate: the exported `ca.crt` when TLS is enabled

4. Enter the SFM Prometheus pod and run the generated
   `victoria_metrics.notes.sfm.hosts_entry` command so its internal vminsert
   service name resolves to the LoadBalancer IP. Repeat the update if the pod
   restarts.

## Verification

Confirm that the generated file contains non-empty vminsert and vmselect
endpoints. Then query VictoriaMetrics through the generated
`victoria_metrics.endpoints.vmselect.query_endpoint` for a metric published by
SFM.

## Next steps

- Use the generated `victoria_metrics.endpoints.vmselect.ui_url` for interactive
  queries.
- Use [Export VictoriaLogs Connection Details](configure_external_victoria_logs.md)
  when SFM or another external system must send logs.

## Troubleshooting

- **No vminsert or vmselect IP is exported:** Assign external IPs to both
  LoadBalancer services and rerun the utility.
- **TLS validation fails:** Use the exported `ca.crt` and the generated service
  hostname rather than substituting an unrelated certificate or name.
- **Remote write stops after a pod restart:** Reapply the generated hosts entry
  inside the SFM Prometheus pod.
