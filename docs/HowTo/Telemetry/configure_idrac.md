# Configure iDRAC Telemetry

## Overview

iDRAC Telemetry deploys one `idrac-telemetry` StatefulSet containing MySQL,
ActiveMQ, a receiver, Kafka pump, and Victoria pump containers. The source
requires both Kafka and VictoriaMetrics collection targets. When a BMC CSV is
provided, the workflow validates BMC reachability, Redfish access, credentials,
firmware, and an iDRAC Datacenter license before enabling collection.

MySQL stores the iDRAC service inventory used by the receiver. It runs as the
`mysqldb` container in the StatefulSet, uses the `idrac_telemetrydb` database,
and is reachable only through the internal `idrac-telemetry-service` headless
service. Its data is stored on a ReadWriteOnce persistent volume.

## Prerequisites

- Complete the common [Telemetry deployment prerequisites](deploy_telemetry.md#prerequisites).
- Provide a BMC CSV with the header `BMC_IP,GROUP_NAME,PARENT`; set its path in
  `idrac_telemetry_configurations.bmc_group_data_path`.
- Ensure the BMCs are reachable from at least one service Kubernetes worker.
  If the worker cannot be reached over SSH, the source falls back to validating
  BMCs from the control-plane VIP.
- Have one common `bmc_username` and `bmc_password` for the BMCs, plus
  `mysqldb_user`, `mysqldb_password`, and `mysqldb_root_password`.

  These credentials are requested only when iDRAC metrics are enabled. They are
  stored in the encrypted project file
  `<OMNIA_DATA_PATH>/telemetry/input/<project>/telemetry_credentials.yml` and
  deployed to the `mysqldb-credentials` Kubernetes Secret.

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
      bmc_group_data_path: "<ORCHESTRATOR_DATA_PATH>/output/<OMNIA_PROJECT_NAME>/bmc_group_data.csv"
      mysqldb_storage: "1Gi"
      oim_bmc_ips:
        oim1: ""
        oim2: ""
    ```

    Resolve `ORCHESTRATOR_DATA_PATH` from `/etc/omnia/omnia.env`; when it is
    unset, use `<OMNIA_DATA_PATH>/orchestrator`. Replace both placeholders
    with their absolute values because environment variables are not expanded
    inside YAML.

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

Use [Verify iDRAC Telemetry](verify_idrac.md) to check the MySQL container,
Secret, PVC, database, and non-sensitive service inventory fields.

## Lifecycle and cleanup

Setting `telemetry_sources.idrac.metrics_enabled: false` and running Telemetry
deployment scales the `idrac-telemetry` StatefulSet to zero replicas. The MySQL
PVC is preserved so the service inventory remains available when iDRAC
telemetry is enabled again.

To remove only the iDRAC Telemetry resources while preserving the MySQL PVC:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run telemetry --tags cleanup_idrac
```

Delete the MySQL PVC only when a complete iDRAC telemetry data reset is
intended:

```bash title="Run on: OIM"
./omnia.sh --run telemetry --tags cleanup_idrac -e Delete_volume=true
```

!!! warning

    `Delete_volume=true` permanently removes the MySQL service inventory.

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
- **MySQL is not ready:** Inspect the `mysqldb` container and the
  `cleanup-mysql-locks` init container by following
  [Verify iDRAC Telemetry](verify_idrac.md).
