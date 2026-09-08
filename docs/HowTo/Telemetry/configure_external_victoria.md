# Export VictoriaMetrics Connection Details

## Overview

The `external_victoria` utility verifies VictoriaMetrics, discovers its
vminsert and vmselect LoadBalancer endpoints, detects whether the
`victoria-tls-certs` Secret exists, and exports the write URL, query URL, UI
URL, and CA certificate when TLS is enabled.

## Prerequisites

- Deploy VictoriaMetrics through the Telemetry workflow.
- Ensure its pods are Running in the `telemetry` namespace.
- Ensure `vminsert-victoria-cluster` and `vmselect-victoria-cluster` have
  LoadBalancer external IPs.
- Ensure the OIM can reach the Kubernetes VIP over root SSH.

## Procedure

1. Run the export utility:

    ```bash title="Run on: OIM"
    ./omnia.sh -r telemetry --tags external_victoria
    ```

2. Read:

    ```text
    <OMNIA_DATA_PATH>/telemetry/output/<project>/external_victoria/external_victoria_connect_details.yml
    ```

3. Use the generated fields for clients:

    - `victoria_metrics.endpoints.vminsert.write_endpoint`
    - `victoria_metrics.endpoints.vmselect.query_endpoint`
    - `victoria_metrics.endpoints.vmselect.ui_url`
    - `victoria_metrics.tls.ca_crt`

    The utility selects `https` when the Victoria TLS Secret exists and `http`
    otherwise.

## Verification

Confirm the output file contains non-empty vminsert and vmselect hosts. When
TLS is enabled, also confirm that `ca.crt` exists in the same output directory.
The utility embeds the current VictoriaMetrics pod status in the generated
file.

## Next steps

- Use the exported values to [connect SFM](configure_sfm.md) or another metrics
  producer.
- Use [Export VictoriaLogs Connection Details](configure_external_victoria_logs.md)
  for log and syslog endpoints written by the same utility.

## Troubleshooting

- **No VictoriaMetrics pods are found:** Enable a metrics route and deploy
  Telemetry.
- **A pod is not Running:** Inspect the VictoriaMetrics pods in the
  `telemetry` namespace before rerunning the export.
- **An external IP is missing:** Assign external IPs to both vminsert and
  vmselect LoadBalancer services.
- **The VIP cannot be reached:** Restore root SSH access from the OIM to the
  configured Kubernetes VIP.

