# Telemetry

## Overview

The Telemetry deployment module deploys and manages Kubernetes workloads that collect HPC
and infrastructure metrics and logs. It supports iDRAC, LDMS, OpenManage
Enterprise (OME), PowerScale, NVIDIA UFM, and VAST sources. Depending on the
configured routes, it deploys Kafka, VictoriaMetrics, VictoriaLogs, and Vector
bridges in the `telemetry` namespace.

Telemetry runs from the Omnia Infrastructure Manager (OIM). Kubernetes actions
run through SSH on the control-plane VIP obtained from the configured
Orchestrator inventory.

```text
sources -> optional Vector bridges -> Kafka / VictoriaMetrics / VictoriaLogs
                                      |
                                      v
                        telemetry_status.yml and connection exports
```

## Prerequisites

| Requirement | Supported value |
|---|---|
| OIM operating system | RHEL or Rocky Linux 10.x |
| Python | 3.12 or later |
| Ansible | 2.20 or later |
| Kubernetes | A deployed service cluster reachable through `kube_vip` |
| Access | Root SSH access from the OIM to `kube_vip` |

Source-specific requirements are listed in each configuration guide. Review
the [Telemetry Input/Output Contract](../../Reference/domain_contracts/telemetry_contract.md)
before configuring the project inputs.

## Procedure

| Task | Use it to |
|---|---|
| [Build Telemetry Container Images](setup_telemetry.md) | Build the iDRAC pump and receiver images and the LDMS image maintained by the Telemetry source. |
| [Deploy the Telemetry Stack](deploy_telemetry.md) | Initialize runtime inputs, validate them, run prechecks, deploy enabled components, and inspect deployment status. |
| [Configure iDRAC Telemetry](configure_idrac.md) | Collect Dell server BMC metrics into Kafka and VictoriaMetrics. |
| [Prepare Worker-to-BMC Network Access](worker_node_vlan_configuration.md) | Verify the worker and Redfish network path required by the iDRAC workflow. |
| [Configure LDMS Telemetry](configure_ldms.md) | Deploy LDMS samplers and Kubernetes aggregator/store components, with an optional Vector-to-VictoriaMetrics bridge. |
| [Configure PowerScale Telemetry](configure_powerscale.md) | Deploy CSM Metrics PowerScale and route metrics to VictoriaMetrics; prepare the VictoriaLogs syslog target when logs are enabled. |
| [Configure UFM Telemetry](configure_ufm.md) | Scrape an existing UFM Prometheus endpoint and prepare optional log ingestion through VLAgent. |
| [Configure VAST Telemetry](configure_vast.md) | Scrape an existing VAST Prometheus endpoint and prepare optional log ingestion through VLAgent. |
| [Configure OME Telemetry](telemetry_from_ome.md) | Route OME Kafka topics through Vector to VictoriaMetrics and VictoriaLogs. |
| [Connect SFM](configure_sfm.md) | Export the VictoriaMetrics connection settings generated for SFM remote write. |
| [Export Kafka Connection Details](configure_external_kafka.md) | Export the native Kafka mTLS endpoint, HTTP Bridge endpoint, and client certificates. |
| [Export VictoriaMetrics Connection Details](configure_external_victoria.md) | Export VictoriaMetrics write/query endpoints and the TLS CA when enabled. |
| [Export VictoriaLogs Connection Details](configure_external_victoria_logs.md) | Export VictoriaLogs write/query endpoints and the VLAgent syslog target. |

The source-specific verification pages repeat the checks independently when a
deployment must be inspected later: [iDRAC](verify_idrac.md),
[LDMS](verify_ldms.md), [OME](verify_ome.md),
[PowerScale](verify_powerscale.md), [UFM](verify_ufm.md),
[VAST](verify_vast.md), and [Vector-LDMS](verify_vector_ldms.md).

### Contract reference

See the [Telemetry Input/Output Contract](../../Reference/domain_contracts/telemetry_contract.md)
for the complete validated input, status, cleanup, and connection-export
contracts.

Telemetry reads these project-scoped runtime inputs:

| Input | Default location |
|---|---|
| `telemetry_config.yml` | `/opt/omnia/telemetry/input/project_default/` |
| `telemetry_storage_config.yml` | `/opt/omnia/telemetry/input/project_default/` |
| `telemetry_packages.yml` | `/opt/omnia/telemetry/input/project_default/` |
| `telemetry_credentials.yml` | Created and encrypted in the same directory when credentials are collected |

`OMNIA_DATA_PATH` and `OMNIA_PROJECT_NAME` change the root and project portions
of these paths. The deployment writes
`<OMNIA_DATA_PATH>/telemetry/output/<project>/telemetry_status.yml`. It records
the overall result, Kubernetes namespace, VIP, package mode, per-sink and
per-source results, bridge results, and LDMS nodes skipped as unreachable.

## Verification

After deployment, inspect:

```bash title="Run on: OIM host"
cat "$OMNIA_DATA_PATH/telemetry/output/$OMNIA_PROJECT_NAME/telemetry_status.yml"
```

Confirm that `overall_status` is `success`, enabled components report
`deployed`, disabled components report `skipped`, and any node listed under
`deploy_unreachable_nodes.ldms` is intentionally unavailable. Use the
source-specific verification page when validating metrics or logs after the
initial deployment.

## Next steps

- Export Kafka or Victoria connection details for external producers and
  consumers when required.
- Use the source-specific configuration pages to add or change a telemetry
  route, then validate and redeploy.
- Preserve `telemetry_status.yml` when collecting evidence for a support case.

## Troubleshooting

- **The Kubernetes VIP is unavailable:** Verify the file selected by
  `cluster_inventory` and restore root SSH access from the OIM.
- **Input validation fails:** Check all three YAML inputs against the schemas
  under `src/telemetry/plugins/module_utils/input_validation/schema/`.
- **A component reports `failed`:** Inspect `/var/log/omnia/telemetry/` and
  the corresponding resources in the `telemetry` namespace.
- **LDMS nodes are skipped:** Check the hostnames under
  `deploy_unreachable_nodes.ldms` and restore their SSH reachability before
  redeploying.
