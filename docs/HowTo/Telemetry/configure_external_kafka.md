# Export Kafka Connection Details

## Overview

The `external_kafka` Telemetry utility verifies the deployed Kafka cluster and
exports its native mTLS endpoint, HTTP Bridge endpoint, cluster CA, and client
certificate and key. Use the native endpoint for Kafka clients such as OME and
the HTTP Bridge endpoint for REST-based validation.

## Prerequisites

- Deploy Kafka through the Telemetry workflow.
- Ensure all Kafka pods in the `telemetry` namespace are Running and Ready.
- Ensure `kafka-kafka-external-bootstrap` and `bridge-bridge-lb` have
  LoadBalancer external IPs and service ports.
- Ensure the OIM can reach the Kubernetes VIP over root SSH.

## Procedure

1. Run the export utility:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run telemetry --tags external_kafka
    ```

    The equivalent command from `src/telemetry` is:

    ```bash title="Run on: OIM"
    ansible-playbook playbooks/telemetry.yml --tags external_kafka
    ```

2. Read the output directory:

    ```text
    <OMNIA_DATA_PATH>/telemetry/output/<project>/external_kafka/
    ├── ca.crt
    ├── user.crt
    ├── user.key
    └── external_kafka_connect_details.yml
    ```

3. Use `kafka.bootstrap_server` and the TLS files for a native Kafka client, or
   use `kafka.bridge.endpoint` for an HTTP client.

4. If OME requires PKCS#12, create the file in the export directory:

    ```bash title="Run on: OIM"
    openssl pkcs12 -export -out user.pfx -inkey user.key -in user.crt
    ```

    Each export run removes and recreates the `external_kafka` directory.
    Generate `user.pfx` after the final export and move it to a secure location
    before rerunning the utility.

## Verification

Confirm that `external_kafka_connect_details.yml` contains non-empty values for
`kafka.bootstrap_server` and `kafka.bridge.endpoint`, and that all three TLS
files exist. The utility fails instead of writing a valid export when Kafka
pods are absent, not Running, not Ready, or either endpoint is unavailable.

## Next steps

- Use the exported native endpoint and certificates to
  [configure OME](telemetry_from_ome.md).
- Protect `user.key` and the generated PKCS#12 file as client credentials.
- Preserve any generated `user.pfx` outside the export directory before
  rerunning the export utility.

## Troubleshooting

- **No Kafka pods are found:** Deploy a source that targets Kafka and rerun the
  Telemetry deployment.
- **Kafka pods are not Running or Ready:** Inspect them with
  `kubectl get pods -n telemetry -l app.kubernetes.io/name=kafka`.
- **An endpoint is empty:** Ensure both Kafka LoadBalancer services have an
  external IP and port.
- **The VIP cannot be reached:** Restore root SSH access from the OIM to the
  configured Kubernetes VIP.
