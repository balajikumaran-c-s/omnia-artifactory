# Verify iDRAC Telemetry

## Overview

Verify the iDRAC StatefulSet, its containers, the Kafka `idrac` topic, and the
Victoria pump endpoint using the same checks as the deployment role.

## Prerequisites

- Complete [Configure iDRAC Telemetry](configure_idrac.md).
- Run commands on the Kubernetes VIP with access to the `telemetry` namespace.

## Procedure

1. Read the StatefulSet readiness and locate its pod:

    ```bash title="Run on: Kubernetes VIP"
    kubectl get statefulset idrac-telemetry -n telemetry
    kubectl get pods -n telemetry -l app=idrac-telemetry
    ```

2. Inspect readiness for every container in the pod:

    ```bash title="Run on: Kubernetes VIP"
    POD=$(kubectl get pods -n telemetry -l app=idrac-telemetry -o jsonpath='{.items[0].metadata.name}')
    kubectl get pod "$POD" -n telemetry \
      -o jsonpath='{range .status.containerStatuses[*]}{.name}={.ready}{"\n"}{end}'
    ```

3. Check the fixed Kafka topic and the Victoria pump metrics endpoint:

    ```bash title="Run on: Kubernetes VIP"
    kubectl wait kafkatopic/idrac -n telemetry --for=condition=Ready --timeout=300s
    kubectl exec -n telemetry "$POD" -c victoria-pump -- \
      wget -qO- http://localhost:2112/metrics
    ```

4. Verify the MySQL Secret and persistent volume claim:

    ```bash title="Run on: Kubernetes VIP"
    kubectl get secret mysqldb-credentials -n telemetry
    kubectl get pvc mysqldb-pvc-idrac-telemetry-0 -n telemetry
    ```

5. Connect to MySQL and inspect only the non-sensitive service inventory
   fields. Use the `mysqldb_user` value from the encrypted Telemetry credential
   file and enter its password when prompted:

    ```bash title="Run on: Kubernetes VIP"
    POD=$(kubectl get pods -n telemetry -l app=idrac-telemetry -o jsonpath='{.items[0].metadata.name}')
    read -rp "MySQL user: " MYSQL_USER
    kubectl exec -it -n telemetry "$POD" -c mysqldb -- \
      mysql -u "$MYSQL_USER" -p idrac_telemetrydb
    ```

    At the MySQL prompt, run:

    ```sql
    SHOW TABLES;
    SELECT ip, serviceType, authType FROM services;
    ```

    !!! caution

        Do not query, copy, or publish the `services.auth` column. It contains
        authentication data used by the iDRAC receiver.

## Verification

At least one StatefulSet replica must be ready, each container must report
`true`, the Kafka topic must be Ready, and the Victoria pump endpoint should
return metrics. The `mysqldb-credentials` Secret must exist, the
`mysqldb-pvc-idrac-telemetry-0` claim must be Bound, and the database query must
return the expected non-sensitive service inventory. Confirm
`sources.idrac.metrics: deployed` in `telemetry_status.yml`.

The status and resource checks do not prove that a BMC is publishing records.
Inspect the `idrac` topic and query VictoriaMetrics for a known BMC metric to
complete end-to-end verification.

## Next steps

- Review `<OMNIA_DATA_PATH>/telemetry/idrac_telemetry_report.yml` when a BMC
  inventory was configured.
- Export [Kafka connection details](configure_external_kafka.md) when external
  access is required.

## Troubleshooting

- **A container is not ready:** Inspect it with
  `kubectl logs -n telemetry "$POD" -c <container-name>`.
- **The topic is absent:** Verify Kafka deployment and that `kafka` remains in
  the iDRAC collection targets.
- **The pump endpoint is pending:** The role retries this check and treats a
  temporarily unavailable endpoint as pending; recheck after data begins to
  flow.
- **MySQL is not ready:** Inspect both initialization and runtime logs:

    ```bash title="Run on: Kubernetes VIP"
    POD=$(kubectl get pods -n telemetry -l app=idrac-telemetry -o jsonpath='{.items[0].metadata.name}')
    kubectl logs -n telemetry "$POD" -c cleanup-mysql-locks
    kubectl logs -n telemetry "$POD" -c mysqldb
    ```

  The initialization container removes stale `.sock` and `.pid` files after an
  ungraceful shutdown. Do not manually delete InnoDB data or lock files without
  a validated recovery plan and a current backup.
