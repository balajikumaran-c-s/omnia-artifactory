# Export VictoriaLogs Connection Details

## Overview

The `external_victoria` utility also discovers VictoriaLogs vlinsert and
vlselect LoadBalancer endpoints and the VLAgent syslog target. These sections
are marked unavailable when their services are not deployed.

## Prerequisites

- Meet the [VictoriaMetrics export prerequisites](configure_external_victoria.md#prerequisites),
  because the utility requires a healthy VictoriaMetrics deployment.
- Deploy VictoriaLogs and VLAgent through a source whose enabled log channel
  targets `victoria_logs`.
- Ensure the VictoriaLogs and VLAgent LoadBalancer services have external IPs.

## Procedure

1. Run the shared Victoria export utility:

    ```bash title="Run on: OIM"
    ./omnia.sh -r telemetry --tags external_victoria
    ```

2. Read the generated connection file:

    ```text
    <OMNIA_DATA_PATH>/telemetry/output/<project>/external_victoria/external_victoria_connect_details.yml
    ```

3. Use its generated values:

    - `victoria_logs.endpoints.vlinsert.write_endpoint` uses port `9481` and
      `/insert/jsonline`.
    - `victoria_logs.endpoints.vlselect.query_endpoint` uses port `9471` and
      `/select/logsql/query`.
    - `victoria_logs.endpoints.vlselect.ui_url` uses port `9471` and
      `/select/vmui`.
    - `vlagent.syslog_endpoint` uses port `514`.

## Verification

Confirm `victoria_logs.available: true` and `vlagent.available: true` in the
generated file and verify their endpoint fields are non-empty. The utility
does not claim that an external producer is sending data; verify ingestion by
querying for records produced by that source.

## Next steps

- Use the generated PowerScale `isi audit` commands when PowerScale log
  collection is enabled.
- Retain the generated connection file as the authoritative endpoint output
  for external log producers.

## Troubleshooting

- **VictoriaLogs is reported unavailable:** Enable a log source targeting
  `victoria_logs`, deploy Telemetry, and confirm the vlinsert service has an
  external IP.
- **VLAgent is reported unavailable:** Inspect the `vlagent` LoadBalancer
  service in the `telemetry` namespace and assign an external IP.
- **The export stops before checking logs:** Restore the required
  VictoriaMetrics deployment; the shared utility validates it first.

