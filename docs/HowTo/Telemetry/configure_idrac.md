# Configure iDRAC Telemetry

## Overview

iDRAC Telemetry deploys one `idrac-telemetry` StatefulSet containing MySQL,
ActiveMQ, a receiver, Kafka pump, and Victoria pump containers. The source
requires both Kafka and VictoriaMetrics collection targets. When a BMC CSV is
provided, the workflow validates BMC reachability, Redfish access, credentials,
firmware, and an iDRAC Datacenter license before enabling collection.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Provide a BMC CSV with the header `BMC_IP,GROUP_NAME,PARENT`; set its path in
  `idrac_telemetry_configurations.bmc_group_data_path`.
- Ensure the BMCs are reachable from at least one service Kubernetes worker.
  If the worker cannot be reached over SSH, the source falls back to validating
  BMCs from the control-plane VIP.
- Have one common `bmc_username` and `bmc_password` for the BMCs, plus
  `mysqldb_user`, `mysqldb_password`, and `mysqldb_root_password`.

## Procedure

1. In the project `telemetry_config.yml`, enable iDRAC and retain both required
   targets:

    ```yaml
    telemetry_sources:
      idrac:
        metrics_enabled: true
        collection_targets:
          - victoria_metrics
          - kafka

    idrac_telemetry_configurations:
      bmc_group_data_path: "/opt/omnia/input/bmc_group_data.csv"
      mysqldb_storage: "1Gi"
      oim_bmc_ips:
        oim1: ""
        oim2: ""
    ```

2. Keep the `idrac_telemetry_storage` resource sections in
   `telemetry_storage_config.yml` and the `images.idrac` entries in
   `telemetry_packages.yml` aligned with the images available to the cluster.

3. Run the Telemetry precheck, then deploy:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags precheck
    ./omnia.sh --run telemetry --tags deploy
    ```

    Enter the requested BMC and MySQL credentials when the credential workflow
    finds an empty value.

## Verification

The deployment itself waits for the StatefulSet, verifies every container,
waits for the `idrac` Kafka topic, and checks the Victoria pump metrics
endpoint. To repeat the principal checks on the VIP, run:

```bash title="Run on: Kubernetes VIP"
kubectl get statefulset idrac-telemetry -n telemetry
kubectl get pods -n telemetry -l app=idrac-telemetry
kubectl wait kafkatopic/idrac -n telemetry --for=condition=Ready --timeout=300s
```

Also confirm `sources.idrac.metrics: deployed` in `telemetry_status.yml`. When
a BMC CSV was configured, review
`<OMNIA_DATA_PATH>/telemetry/idrac_telemetry_report.yml` for activated,
unsupported, invalid, unreachable, and removed BMCs.

These checks confirm the deployed resources. Confirm records in Kafka and
VictoriaMetrics separately to verify end-to-end collection from a BMC.

## Next steps

- Use [Verify iDRAC Telemetry](verify_idrac.md) for the repeatable source checks.
- Use [Export Kafka Connection Details](configure_external_kafka.md) when an
  external client needs the Kafka endpoint and certificates.

## Troubleshooting

- **The configuration is rejected:** Ensure iDRAC is enabled, both targets are
  present, and `mysqldb_storage` is not empty.
- **A BMC is listed as invalid:** Confirm the common BMC credentials, Redfish
  availability, required firmware, and Datacenter license.
- **A BMC is unreachable:** Restore network access from a service worker or the
  control-plane VIP. The workflow selects the first service worker and retries
  the second service worker, when present, before it falls back to the VIP. The
  Telemetry source validates reachability but does not configure site VLANs or
  routes.
- **Kafka or VictoriaMetrics is missing:** Confirm the corresponding sink is
  deployed; both are required by the iDRAC role.
