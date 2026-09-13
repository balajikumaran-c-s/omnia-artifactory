# Deploy OpenLDAP

## Overview

Orchestrator deploys the `omnia_auth` OpenLDAP container on the OIM when the
selected deployment catalog contains an OpenLDAP group. The source derives the
LDAP search base from `SYSTEM_DOMAIN_NAME`, uses `SYSTEM_ADMIN_NIC_IPV4` as the
LDAP server address, collects the OpenLDAP database username and password in
an Ansible Vault file, generates TLS material, and configures LDAP clients
during Slurm provisioning. The standard Kubernetes bolt-on list does not
configure an OpenLDAP client.

OpenLDAP selection is catalog-driven. There is no `ldap_enabled` input and no
`deploy_openldap` top-level tag.

## Prerequisites

- Complete the [OpenCHAMI deployment prerequisites](deploy_openchami.md).
- Ensure the catalog used by Orchestrator contains the required OpenLDAP group.
  Orchestrator derives `openldap_support` when a case-sensitive catalog group
  name contains the lowercase text `openldap`.
- Configure `SYSTEM_DOMAIN_NAME` as an FQDN containing at least one dot. The
  source converts each domain component to the LDAP search base; for example,
  `omnia.cluster` becomes `dc=omnia,dc=cluster`.
- Configure `security_config.yml` in the project input directory with
  `ldap_connection_type: "TLS"` or `ldap_connection_type: "SSL"`. Provisioned
  Slurm and login clients use StartTLS on TCP 389 for `TLS` and LDAPS on TCP
  636 for `SSL`. The `omnia_auth` Quadlet publishes both ports.
- Make the `docker.io/dellhpcomniaaisolution/omnia_auth:1.2` container image
  available to Podman on the OIM.

## Procedure

1. Confirm that the catalog path in `orchestrator_config.yml` points to the
   catalog selected for this deployment. Leave `catalog_file_path` empty to use
   `CATALOG_FILE_PATH` or
   `$OMNIA_DATA_PATH/catalog/catalog_rhel.json`.

2. Set the connection type in the staged input file:

    ```yaml title="security_config.yml"
    ldap_connection_type: "TLS"
    ```

3. Run the precheck. When OpenLDAP is enabled and its credential file
   is absent, the precheck reports that the credentials will be collected in
   the prepare phase.

    ```bash title="Run on: OIM"
    cd src/main
    ./omnia.sh --run orchestrator --tags precheck
    ```

4. Run the `prepare` phase and supply
   `openldap_db_username` and `openldap_db_password` when prompted. The
   credential role stores them in `orchestrator_credentials.yml` in the active
   project's Orchestrator input directory and encrypts the file with
   `.orchestrator_credentials_key`.

    ```bash title="Run on: OIM"
    ./omnia.sh --run orchestrator --tags prepare
    ```

   Credential collection prompts only for values that are empty. Provisioning
   password and BMC username/password are always required; Slurm, OpenLDAP,
   and PowerScale CSI credentials are added when those features are selected.
   Every password prompt requires confirmation. Existing non-empty values are
   retained, so rotate one by editing the Vault-encrypted file with its matching
   key rather than expecting `prepare` to prompt again.

   The phase creates `$OMNIA_DATA_PATH/auth/config`, `tls_certs`, `data`, and
   `init`; generates `slapd.conf`, `bootstrap.ldif`, and a certificate; creates
   `/etc/containers/systemd/omnia_auth.container`; and enables and starts the
   `omnia_auth` systemd service.

## Verification

Run the source-defined deployment health check and inspect the container:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run orchestrator --tags validate-deployment
podman ps --filter name=omnia_auth
systemctl status omnia_auth
```

The health phase skips OpenLDAP when the catalog does not enable it. When it is
enabled, validation fails unless the `omnia_auth` container exists and its
state is `running`.

## Next steps

- Continue with [Provision Nodes](provision_nodes.md). The default Slurm
  provisioning path runs the OpenLDAP client configuration when the catalog
  enables OpenLDAP. Kubernetes does not include OpenLDAP in its default
  bolt-on list; the standard input template does not expose a supported
  Kubernetes OpenLDAP override.
- Use [Deploy Slurm](deploy_slurm.md) or
  [Deploy Kubernetes](deploy_kubernetes.md) for service-specific inputs.

## Troubleshooting

**OpenLDAP is skipped unexpectedly**

Confirm `catalog_file_path` and verify that the selected catalog contains a
group name with the lowercase text `openldap`. The `precheck`, `credentials`,
`prepare`, `deploy`, and `validate-deployment` flows fail when the resolved
catalog is missing or invalid; they do not silently disable OpenLDAP.

**The deployment reports missing credentials**

Run the credential phase, then retry deployment:

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run orchestrator --tags credentials
./omnia.sh --run orchestrator --tags deploy
```

**The `omnia_auth` container is not running**

Inspect the directory and log named in the source validation message:

```bash title="Run on: OIM"
ls -la "$OMNIA_DATA_PATH/auth"
podman logs omnia_auth
systemctl status omnia_auth
```

Then rerun `./omnia.sh --run orchestrator --tags deploy` from `src/main`.
