# Product and Subsystem Security

## Security Controls Map

![Security Controls Map](../assets/images/SecurityControlMap.png)

!!! note

    Omnia supports NFS configured on the following external storage solutions for HPC cluster data storage:

    - Dell PowerVault (iSCSI)
    - Dell PowerScale (CSI)
    - VAST (NFS)

    Each storage system may require specific authentication credentials and configurations. Refer to the respective storage integration documentation for detailed setup instructions.

Omnia performs bare metal configuration to enable AI/HPC workloads. It uses
Ansible playbooks to perform installations and configurations. iDRAC is
supported for provisioning bare metal servers. Omnia enables provisioning of
clusters via PXE using a mapping file **(Mandatory)** to dictate IP
address/MAC mapping.

Omnia can be installed via CLI only. Slurm and Kubernetes are deployed and
configured on the cluster. When a catalog group name contains the lowercase
text `openldap`, Orchestrator deploys OpenLDAP for supported Slurm and login
functional groups.

To perform these configurations and installations, a secure SSH channel is
established between the management node and the following entities:

- `slurm_control_node`
- `slurm_node`
- `login_node`
- `service_kube_control_node`
- `service_kube_node`

## Authentication

Omnia adheres to a subset of the specifications of NIST 800-53 and NIST 800-171 guidelines on the OIM and login node.

Omnia does not have its own authentication mechanism because bare metal installations and configurations take place using root privileges. Post the execution of Omnia, third-party tools are responsible for authentication to the respective tool.

## Cluster Authentication Tool

For centralized authentication, Orchestrator can deploy OpenLDAP, an open
source directory service for Linux networked environments. Selection is
catalog-driven, and the current provisioning templates configure OpenLDAP
clients on supported Slurm control, compute, and login functional groups. The
Kubernetes provisioning path does not configure an OpenLDAP client. Site
administrators remain responsible for user and group management.

!!! note

    Omnia does not configure OpenLDAP users or groups.

## Authentication Types and Setup

### Key-Based authentication

**Use of SSH authorized_keys**

A password-less channel is created between the management station and compute nodes using SSH authorized keys. This is explained in the [Security Controls Map](#security-controls-map).

## Login Security Settings

Users provide credentials to the domain that owns the corresponding service.
The domain stores them in an encrypted Ansible Vault under its project input
directory. For iDRAC Telemetry, the credentials are stored in
`<OMNIA_DATA_PATH>/telemetry/input/<project>/telemetry_credentials.yml` and the
Vault key is stored in `.telemetry_credentials_key`. The MySQL credentials are
requested only when iDRAC metrics are enabled.

1. iDRAC/BMC (Username / Password)
2. Provisioning OS (Password)
3. Slurm database (`slurm_db_password`)
4. DockerHub (Username / Password)
5. OpenLDAP (`openldap_db_username`, `openldap_db_password`)
6. Telemetry (`mysqldb_user`, `mysqldb_password`, `mysqldb_root_password`)
7. Minio S3 bucket (Password)
8. Pulp (Password)
9. CSI PowerScale credentials (Username / Password)
10. LDMS Sampler (Password)
11. Postgres (`postgres_user`, `postgres_password`)
12. GitLab (`gitlab_root_password`)
13. OME Discovery (`ome_username`, `ome_password`)
14. UFM Telemetry (`ufm_username`, `ufm_password`)
15. VAST Telemetry (`vast_username`, `vast_password`)















