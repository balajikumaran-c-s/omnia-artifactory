# Telemetry Input/Output Contract

**Deployment module**: Telemetry | **CLI identifier**: `telemetry`

## Input contract

Telemetry reads its project inputs from:

```text
$OMNIA_DATA_PATH/telemetry/input/$OMNIA_PROJECT_NAME/
```

`TELEMETRY_DATA_PATH` can replace
`$OMNIA_DATA_PATH/telemetry`. The defaults resolve to
`/opt/omnia/telemetry/input/project_default/`.

The current deployment loader still reads the `project_default` input
directory, regardless of the project selected during validation, and
initialization and deployment do not consistently honor `TELEMETRY_DATA_PATH`.
Use the default project and data root until these source limitations are
corrected.

| Input | Required | Purpose |
|---|---|---|
| `telemetry_config.yml` | Yes | Selects the cluster inventory, sources, routes, bridges, sinks, and source-specific settings. |
| `telemetry_storage_config.yml` | Yes | Defines storage and resource settings for the selected telemetry components. |
| `telemetry_packages.yml` | Yes | Defines online or offline package sources, cluster mounts, images, charts, Git repositories, and Python packages. |
| `telemetry_credentials.yml` | Conditional | Ansible Vault-encrypted credentials collected for enabled sources and sinks. |
| `.telemetry_credentials_key` | With the credential file | Vault password file for the encrypted credentials. |
| Cluster inventory | Yes | File selected by `cluster_inventory`; supplies the service Kubernetes VIP and Slurm node groups. |

The YAML inputs are validated against the schemas under
`plugins/module_utils/input_validation/schema/` and by the corresponding
cross-field validators.

### `telemetry_config.yml`

| Field | Requirement | Purpose |
|---|---|---|
| `cluster_inventory` | Required, non-empty path | Selects the Ansible inventory containing `kube_vip_group` and any Slurm nodes used by LDMS. |
| `telemetry_sources.<source>.metrics_enabled` | Source-dependent | Enables metrics for `idrac`, `ldms`, `powerscale`, `ufm`, `vast`, or `ome`. |
| `telemetry_sources.<source>.logs_enabled` | Where supported | Enables logs for PowerScale, UFM, VAST, or OME. |
| `telemetry_sources.<source>.collection_targets` | Required for configured sources | Routes a source to its schema-supported sinks. |
| `telemetry_bridges.vector_ldms` | Optional | Routes LDMS data from Kafka to VictoriaMetrics. |
| `telemetry_bridges.vector_ome` | Optional | Routes OME metrics or logs from Kafka to VictoriaMetrics or VictoriaLogs. |
| Source-specific configuration | Conditional | Supplies endpoints, ports, inventory paths, and other fields required by enabled sources. |

For iDRAC metrics, `idrac_telemetry_configurations` supplies the following
MySQL and inventory settings:

| Field | Requirement | Purpose |
|---|---|---|
| `bmc_group_data_path` | Required when iDRAC metrics are enabled | Selects the BMC inventory CSV consumed by the iDRAC source. |
| `mysqldb_storage` | Required, non-empty; default `1Gi` | Sets the requested capacity of the MySQL PVC. |
| `oim_bmc_ips` | Optional | Adds OIM BMC addresses to the service inventory when configured. |

The encrypted `telemetry_credentials.yml` contains `bmc_username`,
`bmc_password`, `mysqldb_user`, `mysqldb_password`, and
`mysqldb_root_password` when iDRAC metrics are enabled. The deployment renders
the MySQL values into the `mysqldb-credentials` Kubernetes Secret.

PowerScale, UFM, and VAST are imported by the root deployment only when their
metrics channel is enabled. Logs for these external systems require manual
configuration to forward syslog to VLAgent; a logs-only source configuration
is not supported by the current root workflow. OME is an external Kafka
producer, and SFM is an external VictoriaMetrics producer rather than a
Telemetry source role.

### `telemetry_storage_config.yml`

Storage sections are required when their corresponding components are selected.
They include Kafka, VictoriaMetrics, VictoriaLogs, Vector, iDRAC, LDMS,
PowerScale, UFM, and VAST storage or resource settings as applicable.

### `telemetry_packages.yml`

| Field | Requirement | Purpose |
|---|---|---|
| `install_mode` | Optional; default `offline` | Selects `offline` or `online` package resolution. |
| `repo_url` | Required in offline mode | Base Pulp content URL. |
| `k8s_cluster_mount` | Required | Existing mount used to stage Telemetry content for Kubernetes nodes. |
| `slurm_cluster_mount` | Required | Existing mount used for LDMS content on Slurm nodes. |
| `container_registry` | Optional | Overrides the registry prefix for air-gapped deployment. |
| Package maps | Required as consumed | Image, Helm chart, Git repository, and Python package definitions used by the selected components. |

## Output contract

### Deployment and cleanup status

Telemetry writes one authoritative status file:

```text
$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/telemetry_status.yml
```

When `TELEMETRY_DATA_PATH` is set, the output is written below that root
instead.

| Field | Purpose |
|---|---|
| `domain` | Records `telemetry`. |
| `type` | Identifies a `deploy` or `cleanup` result. |
| `project_name` | Active project. |
| `overall_status` | `success`, `failed`, or `partial`. |
| `generated_at` | Generation timestamp. |
| `namespace` | Kubernetes namespace, normally `telemetry`. |
| `kube_vip` | Kubernetes control-plane VIP used by the workflow. |
| `packages` | Deployment package mode and repository URL. |
| `sinks` | Kafka, VictoriaMetrics, and VictoriaLogs results. |
| `sources` | Per-source metrics and, where supported, logs results. |
| `bridges` | Vector-LDMS and Vector-OME results. |
| `deploy_unreachable_nodes.ldms` | LDMS nodes skipped during deployment because they were unreachable. |

Component deployment values are `deployed`, `failed`, or `skipped`.

The root deployment currently calls the sink playbook without a derived sink
selection, so Kafka, VictoriaMetrics, and VictoriaLogs are deployed by
default. Status values report the state evaluated by the deployment workflow;
they do not prove end-to-end ingestion. In particular, PowerScale and UFM log
status reflects VLAgent availability, and the current VAST summary does not
consume its source-specific component check. Verify actual records in the
selected sink.

Cleanup rewrites the same file with `type: cleanup` and adds
`cleanup_components`, `cleanup_unreachable_nodes`, and a `volumes` block.
The `Delete_volume` or `delete_volume` boolean extra variable controls
whether persistent volume claims are deleted or preserved. The cleanup
workflow preserves `telemetry_status.yml` as the last-known result.

### Connection exports

| Tag | Output |
|---|---|
| `external_kafka` | `<TELEMETRY_DATA_PATH>/output/<project>/external_kafka/external_kafka_connect_details.yml`, `ca.crt`, `user.crt`, and `user.key`. |
| `external_victoria` | `<TELEMETRY_DATA_PATH>/output/<project>/external_victoria/external_victoria_connect_details.yml` and `ca.crt` when TLS is enabled. The YAML contains available VictoriaMetrics, VictoriaLogs, and VLAgent endpoints. |

These utilities fail when their required deployment or endpoint state is not
available instead of presenting an incomplete export as valid.

The domain supports setup, validation, precheck, deployment, cleanup, and the
two external connection exports. The `upgrade` and `rollback` operations are
placeholders in the current source and do not perform lifecycle changes.

## iDRAC MySQL runtime contract

MySQL is an implementation component of the Telemetry domain; it is not an
independent deployment domain.

| Resource | Runtime contract |
|---|---|
| StatefulSet | `idrac-telemetry` in the `telemetry` namespace, with one replica. |
| Container | `mysqldb`, using the MySQL image selected by `images.idrac.mysql`; the current default is `docker.io/library/mysql:9.7.2`. |
| Service | Internal headless service `idrac-telemetry-service`, with ports 3306 and 33060. It is not exported as a customer-facing MySQL endpoint. |
| Database | `idrac_telemetrydb`; the `services` table stores the iDRAC service inventory consumed by the receiver. |
| Secret | `mysqldb-credentials`, generated from the encrypted Telemetry credential file. |
| Persistent storage | `mysqldb-pvc-idrac-telemetry-0`, requested as ReadWriteOnce using `mysqldb_storage`. |
| Recovery initialization | The `cleanup-mysql-locks` init container removes stale `.sock` and `.pid` files after an ungraceful shutdown. |

Disabling iDRAC metrics scales the StatefulSet to zero replicas and preserves
the MySQL PVC. `cleanup_idrac` removes the source resources but also preserves
the PVC by default. Passing `Delete_volume=true` deletes the PVC and permanently
removes the stored service inventory.

The `services.auth` column contains authentication data used by the receiver.
Operational verification must not print or publish this column.

## Related documentation

- [Telemetry](../../HowTo/Telemetry/index.md)
- [Deploy the Telemetry stack](../../HowTo/Telemetry/deploy_telemetry.md)
- [Export Kafka connection details](../../HowTo/Telemetry/configure_external_kafka.md)
- [Export VictoriaMetrics connection details](../../HowTo/Telemetry/configure_external_victoria.md)
- [Export VictoriaLogs connection details](../../HowTo/Telemetry/configure_external_victoria_logs.md)
- [Telemetry configuration](../Configuration/telemetry_config.md)
