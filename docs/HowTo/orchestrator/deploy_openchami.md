# Deploy OpenCHAMI

## Overview

Orchestrator deploys OpenCHAMI on the Omnia Infrastructure Manager (OIM) as
containerized services managed by `openchami.target`. The services used by the
source provisioning workflow include SMD for node state, boot-service for boot
parameters, metadata-service for cloud-init data, tokensmith, ACME certificate
deployment, HAProxy, and the S3 endpoint supplied by Image Build Manager.

The top-level Orchestrator entry point deploys OpenCHAMI and conditionally
deploys OpenLDAP in the same `prepare` or `deploy` phase. It does not provide a
`deploy_openchami` tag.

### OpenCHAMI components

| Component | Purpose |
|---|---|
| `openchami.target` | Manages the OpenCHAMI systemd service group on the OIM. |
| SMD | Stores node, component, and functional-group state. |
| boot-service | Stores the kernel, initrd, root image, and boot parameters used by provisioned nodes. |
| metadata-service | Serves cloud-init and hostname metadata to provisioned nodes. |
| tokensmith | Generates access tokens for authenticated OpenCHAMI API operations. |
| ACME and local CA | Generate and deploy certificates used by the OpenCHAMI endpoints. |
| HAProxy | Provides the TLS entry point for OpenCHAMI APIs. |
| PostgreSQL | Stores persistent SMD service data. |
| CoreDHCP | Provides DHCP and PXE information on the configured admin networks. |
| coresmd and CoreDNS | Provide SMD-backed cluster DNS when `dns_enabled` is `true`. |

The components installed by the OpenCHAMI package can vary by package version.
Use `systemctl list-dependencies openchami.target` to view the deployed service
set on the OIM.

## Prerequisites

- Run the workflow on the OIM with root or equivalent privileges.
- Ensure `/etc/omnia/omnia.env` has been created and the configured system
  hostname, domain, and admin IP match the OIM. `SYSTEM_DOMAIN_NAME` must be a
  non-empty domain and `SYSTEM_ADMIN_NIC_IPV4` must be assigned locally.
- Run Repo Manager successfully. Orchestrator requires `repo_status.yml` to
  report `overall_status: success` and provide a valid Pulp public certificate.
- Run Image Build Manager successfully. Its `build_status.yml` must contain a
  reachable S3 endpoint and images for every functional group in the PXE
  mapping.
- Copy the discovery mapping to the Orchestrator project input directory. The
  mapping must contain the required uppercase columns and unique service tags,
  hostnames, and admin IP addresses.

## Procedure

1. From the Orchestrator source directory, initialize the module. This installs
   its Python and Ansible dependencies, creates runtime directories, and stages
   the input templates without overwriting existing project files unless you
   approve the prompt.

    ```bash title="Run on: OIM"
    cd /omnia/src/orchestrator
    ./domain-init.sh
    ```

2. Edit the project inputs under
   `$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/`:

   - In `orchestrator_config.yml`, set `pxe_mapping_file_path` to the mapping
     CSV. Set `image_build_manager_output_path`, `repo_manager_output_path`, or
     `catalog_file_path` only when their files are not in the default project
     locations.
   - In `network_spec.yml`, configure the OIM admin NIC, admin subnet, OIM admin
     IP, router, DHCP dynamic range, and any DNS, NTP, InfiniBand, or additional
     subnet data used by the cluster.
   - Keep `language: "en_US.UTF-8"`. Set `default_lease_time` to a positive
     number of seconds.

3. Validate the input files, then run the prerequisite checks.

    ```bash title="Run on: OIM"
    ansible-playbook playbooks/orchestrator.yml --tags validate
    ansible-playbook playbooks/orchestrator.yml --tags precheck
    ```

4. Run the `prepare` phase. It collects missing provisioning and BMC
   credentials, deploys OpenCHAMI, conditionally deploys catalog-selected
   OpenLDAP, and runs both readiness gates.

    ```bash title="Run on: OIM"
    ansible-playbook playbooks/orchestrator.yml --tags prepare
    ```

   After the initial preparation, use `--tags deploy` to retry the OpenCHAMI
   and conditional OpenLDAP deployment and its health checks without running
   the credential collection phase.

## Verification

Run the source-defined deployment health checks:

```bash title="Run on: OIM"
ansible-playbook playbooks/orchestrator.yml --tags validate-deployment
```

The check succeeds only when `openchami.target` is active, the authenticated
SMD readiness endpoint responds, boot-service and metadata-service respond,
tokensmith and ACME are active, and the configured S3 health endpoint is
reachable.

You can also inspect the systemd target directly:

```bash title="Run on: OIM"
systemctl status openchami.target
systemctl list-dependencies openchami.target
```

## Next steps

- If the catalog enables OpenLDAP, review [Deploy OpenLDAP](deploy_openldap.md).
- Continue with [Provision Nodes](provision_nodes.md) to register functional
  groups and create boot and cloud-init configuration.

## Troubleshooting

**A required Repo Manager output or certificate is missing**

Run Repo Manager again or set `repo_manager_output_path` to its successful
`repo_status.yml`. The file must contain `cluster_os_type`, a `repositories`
mapping, and `repo_manager.certificates.server_crt`; the referenced certificate
must exist.

**A functional-group image cannot be found**

Run Image Build Manager for the missing functional group and confirm that its
successful `build_status.yml` contains the corresponding kernel, initrd, and
root image. If `kernel_version_override` is set, the requested kernel must exist
in S3; clear the setting to let Orchestrator select the latest available image.

**OpenCHAMI is not ready**

Use the checks emitted by the provisioning role, then rerun the deployment:

```bash title="Run on: OIM"
systemctl status openchami.target
journalctl -u openchami.target -n 50
systemctl status smd boot-service metadata-service
ansible-playbook playbooks/orchestrator.yml --tags deploy
```

Review `/var/log/omnia/orchestrator/orchestrator.log` for the failed Ansible
task.
