# Upgrade Orchestrator

## Overview

The Orchestrator `upgrade` workflow runs the OpenCHAMI and OpenLDAP component
upgrade playbooks in sequence. The current OpenCHAMI workflow is designed for
the migration from `0.1.7-1` to `0.2.0-1`. It records the installed package
version, creates a pre-upgrade backup, removes legacy pre-Fabrica services when
present, pulls the configured OpenCHAMI images, restarts the services, and
checks their health.

When the `omnia_auth` container is deployed, the OpenLDAP workflow pulls
`docker.io/dellhpcomniaaisolution/omnia_auth:1.2`, restarts the service, and
checks the container and LDAP endpoint. It skips OpenLDAP when the container
is not deployed.

!!! warning

    OpenCHAMI and OpenLDAP rollback are not supported in this release. The
    `rollback` tag enters reserved workflows that intentionally stop with a
    `ROLLBACK NOT SUPPORTED` error. The OpenCHAMI pre-upgrade backup does not
    make the rollback workflow operational.

## Prerequisites

- Run the workflow from the OIM with the project environment used for the
  existing Orchestrator deployment.
- Provide a successful Repository Manager `repo_status.yml` and its referenced
  Pulp certificate. The top-level Orchestrator setup validates this contract
  before the `upgrade` workflow starts.
- Confirm OpenCHAMI is healthy and preserve an external backup of required
  service and project data.
- Ensure the OIM can pull the OpenCHAMI and `omnia_auth:1.2` container images.
- Preserve the Orchestrator credential file and Vault key.
- Plan a maintenance window because the workflow stops legacy services and
  restarts OpenCHAMI and OpenLDAP.

## Procedure

1. Initialize the shared environment if it is not already available:

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --setup-venv
    ```

2. Run the component upgrade workflows:

    ```bash title="Run on: OIM"
    ./omnia.sh --run orchestrator --tags upgrade
    ```

   OpenCHAMI writes its pre-upgrade backup below
   `$OMNIA_DATA_PATH/openchami/backups/pre_upgrade_<timestamp>/`. The backup
   contains available OpenCHAMI configuration, SMD records, work-directory
   data, and `backup_metadata.yml`. An
   `$OMNIA_DATA_PATH/.data/upgrade_in_progress.lock` file protects the active
   migration and is removed after successful OpenCHAMI health verification.

3. If node configuration must be refreshed after the component upgrade, rerun
   provisioning:

    ```bash title="Run on: OIM"
    ./omnia.sh --run orchestrator --tags provision
    ```

## Verification

Verify the OpenCHAMI package and services:

```bash title="Run on: OIM"
rpm -q openchami
systemctl is-active openchami.target
/usr/bin/ochami smd service status
systemctl is-active boot-service
```

When OpenLDAP is deployed, confirm its image and service state:

```bash title="Run on: OIM"
podman inspect omnia_auth --format '{{.ImageName}}'
systemctl is-active omnia_auth
podman exec omnia_auth ldapsearch -x -H ldap://localhost -b "" -s base namingContexts
```

The image name must contain the `1.2` tag and both services must report an
active or running state.

## Next steps

- Run [Provision Nodes](provision_nodes.md) when node-side configuration must
  be regenerated.
- Review the project outputs and OpenCHAMI service state before returning the
  cluster to production use.

## Troubleshooting

- **The upgrade lock remains after a failure:** Preserve the backup and logs,
  correct the failed service or image operation, and rerun the idempotent
  upgrade workflow. Do not remove the lock merely to bypass a failed upgrade.
- **OpenLDAP is skipped:** The workflow skips it when the `omnia_auth`
  container is not deployed. Use [Deploy OpenLDAP](deploy_openldap.md) when
  the catalog requires it.
- **A rollback command fails:** This is expected in the current release.
  Restore service state through an approved recovery procedure rather than
  invoking the reserved rollback tag.
- Review `/var/log/omnia/orchestrator/orchestrator.log` and the OpenCHAMI or
  `omnia_auth` service logs for the failing task.
