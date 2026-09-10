
# Log Management

Omnia maintains logs across playbook executions, container operations, and cluster activities. These logs are essential for monitoring, debugging, and troubleshooting deployments across the OIM, Kubernetes, and Slurm environments.

!!! warning

    Do not delete log files or the directories they reside in.

## Log locations

!!! note

    All log paths referenced in this section are on the OIM host filesystem.

### Playbook logs

Each domain writes its Ansible execution log to the OIM host under
`/var/log/omnia/<domain>/`:

| Log File | Purpose | Playbook |
| --- | --- | --- |
| `/var/log/omnia/discovery/discovery.log` | Discovery | `discovery.yml` |
| `/var/log/omnia/repo_manager/repo_manager.log` | Repo Manager | `repo_manager.yml` |
| `/var/log/omnia/image_build_manager/image_build_manager.log` | Image Build Manager | `image_build_manager.yml` |
| `/var/log/omnia/orchestrator/orchestrator.log` | Orchestrator | `orchestrator.yml` |
| `/var/log/omnia/telemetry/telemetry.log` | Telemetry | `telemetry.yml` |
| `/var/log/omnia/utils/utils.log` | Utils | `utils.yml` |

### Domain and service logs

| Location | Purpose |
| --- | --- |
| `<OMNIA_DATA_PATH>/orchestrator/log/openchami/` | OpenCHAMI logs |
| `<OMNIA_DATA_PATH>/repo_manager/log/` | Repository processing and Pulp logs |
| `<OMNIA_DATA_PATH>/image_build_manager/log/<OMNIA_PROJECT_NAME>/` | Image build logs |
| `<OMNIA_DATA_PATH>/build_stream_root/artifacts/<job_id>/` | Build Stream job artifacts and results |

!!! note

    BuildStreaM and GitLab log paths are available inside the BuildStreaM container.

### Slurm logs

On Slurm cluster nodes, logs are stored in standard Slurm log directories:

| Path | Description |
| --- | --- |
| `/var/log/slurm/slurmctld.log` | Slurm controller daemon log (on the control node). |
| `/var/log/slurm/slurmd.log` | Slurm compute daemon log (on each compute node). |
| `/var/log/slurm/slurmdbd.log` | Slurm database daemon log (job accounting). |

## Viewing Podman container logs on OIM

1. List all containers running on the OIM:

    ```bash title="Run on: OIM host"
    podman ps -a
    ```

    ```text title="Expected output"
    CONTAINER ID   IMAGE                                                        COMMAND                  CREATED        STATUS        PORTS                                        NAMES
    222d8d96554b   localhost/omnia_auth:1.0                                     /bin/sh -c mkdi…         2 days ago     Up 2 days     0.0.0.0:389->389/tcp, 0.0.0.0:636->636/tcp   omnia_auth
    42515184e9ba   docker.io/pgsty/minio:RELEASE.2026-04-17T00-00-00Z           server /data –co…        2 days ago     Up 2 days     0.0.0.0:9000-9001->9000-9001/tcp             minio-server
    5bbce8efdc27   docker.io/pulp/pulp:3.80                                     /init                    2 days ago     Up 2 days     0.0.0.0:2225->2225/tcp, 80/tcp               pulp
    d0b76ac340c7   docker.io/library/postgres:16                                postgres                 2 days ago     Up 2 days     5432/tcp                                     omnia_postgres
    0608aa923b7c   docker.io/dellhpcomniaaisolution/omnia_build_stream:1.1                               2 days ago     Up 2 days                                                  omnia_build_stream
    251709e17ec2   docker.io/library/registry:3.1.0                             /etc/distribution…       2 days ago     Up 2 days     0.0.0.0:5000->5000/tcp                       registry
    992a9ca91d5d   docker.io/library/postgres:11.5-alpine                       postgres                 2 days ago     Up 2 days     5432/tcp                                     postgres
    b731b88c87ff   ghcr.io/openchami/local-ca:v0.2.6                            /step-ca.sh              2 days ago     Up 2 days     9000/tcp                                     step-ca
    ec2a4422424d   docker.io/oryd/hydra:v2.3                                    serve -c /etc/con…       2 days ago     Up 2 days                                                  hydra
    0751a15ca614   ghcr.io/openchami/opaal:v0.3.12                              /opaal/opaal serv…       2 days ago     Up 2 days                                                  opaal-idp
    72e566933565   ghcr.io/openchami/smd:v2.19.3                                /smd                     2 days ago     Up 2 days     27779/tcp                                    smd
    e95d50f8da14   ghcr.io/openchami/opaal:v0.3.12                              /opaal/opaal logi…       2 days ago     Up 2 days                                                  opaal
    3fac48412fe9   ghcr.io/openchami/bss:v1.32.2                                /bin/sh -c /usr/l…       2 days ago     Up 2 days     27778/tcp                                    bss
    81f488e6b5cb   ghcr.io/openchami/cloud-init:v1.3.0                          /usr/local/bin/cl…       2 days ago     Up 2 days                                                  cloud-init-server
    7e970ad459ca   cgr.dev/chainguard/haproxy:latest                            haproxy -f /usr/l…       2 days ago     Up 2 days     0.0.0.0:8081->80/tcp, 0.0.0.0:8443->443/tcp  haproxy
    5f5028bab1cd   ghcr.io/openchami/coresmd:v0.4.3                             /coredhcp                2 days ago     Up 2 days                                                  coresmd-coredhcp
    354f3305a7b8   ghcr.io/openchami/coresmd:v0.4.3                             /coredns                 2 days ago     Up 2 days                                                  coresmd-coredns
    ```

    !!! note

        Container IDs and image versions will vary on every system.

2. View logs from a specific container:

    ```bash title="Run on: OIM host"
    podman logs <container_name>
    ```

3. If the container is managed as a systemd service, view logs with:

    ```bash title="Run on: OIM host"
    journalctl -xeu <container_name>
    ```

## Viewing Kubernetes pod logs

1. List all namespaces and pods:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get pods -A
    ```

2. Get the list of containers in a pod:

    ```bash title="Run on: Kubernetes control plane"
    kubectl get pods <pod_name> -o jsonpath='{.spec.containers[*].name}'
    ```

3. View logs for a specific container:

    ```bash title="Run on: Kubernetes control plane"
    kubectl logs <pod_name> -n <namespace> -c <container_name>
    ```

## Cluster log collection

Use the current Utils `collect` workflow to gather the source-defined
Kubernetes and Slurm log paths and create a timestamped archive with
`metadata.json`. See [Collect Cluster Logs](collect_cluster_logs.md) for the
supported input file, command, output location, verification, and cleanup
procedure.

!!! info

    - [General Troubleshooting](../Troubleshooting/general.md) -- Uses logs as a primary diagnostic tool.
    - [Best Practices Checklist](best_practices_checklist.md) -- Storage and maintenance best practices.
















