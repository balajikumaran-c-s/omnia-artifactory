# Network Security

Omnia configures the firewall as required by the third-party tools to enhance security by restricting inbound and outbound traffic to the TCP and UDP ports.

## Network Exposure

Omnia uses port 22 for SSH connections, same as Ansible.

## Firewall Settings

Omnia configures the following ports for use by third-party tools installed by Omnia.

### Host Port Requirements

| Port Number | Protocol | Service | Type of Node |
|-------------|----------|---------|--------------|
| 22 | TCP | SSH | All Nodes |
| 2049 | TCP/UDP | NFS Server | Manager (OIM) |
| 111 | TCP/UDP | RPC Bind | Manager (OIM) |
| 20048 | TCP/UDP | NFS mountd | Manager (OIM) |
| 123 | UDP | NTP | Manager (OIM) |
| 53 | TCP/UDP | DNS – Cluster | Manager (OIM) |
| 9153 | TCP | DNS Metrics | Manager (OIM) |
| 53 | TCP/UDP | DNS – Podman Internal | Manager (OIM) |
| 67 | UDP | DHCP | Manager (OIM) |
| 68 | UDP | DHCP BootPC | Manager (OIM) |
| 69 | UDP | TFTP | Manager (OIM) |
| 3702 | UDP | WS-Discovery (OS) | Manager (OIM) |
| 5353 | UDP | mDNS (OS) | Manager (OIM) |
| 631 | TCP | CUPS printing (OS) | Manager (OIM) |

### Podman Container Port Requirements

| Port | Protocol | Service Name | Type of Node |
|---|---|---|---|
|2225|TCP|Pulp Content Service|Manager (OIM)|
|5000|TCP|OCI Registry|Manager (OIM)|
|9000|TCP|MinIO S3 API|Manager (OIM)|
|9001|TCP|MinIO Console|Manager (OIM)|

### Kubernetes Port Requirements

| Port Number | Protocol | Service | Type of Node |
|---|---|---|---|
|6443|TCP|Kubernetes API server|Manager|
|2379-2380|TCP|etcd server client API|Manager|
|10251|TCP|Kube-scheduler|Manager|
|10252|TCP|Kube-controller manager|Manager|
|10250|TCP|Kubelet API|Compute|
|30000-32767|TCP|NodePort services|Compute|
|5473|TCP|Calico services|Manager/Compute|
|179|TCP|Calico services|Manager/Compute|
|4789|UDP|Calico services|Manager/Compute|
|8285|UDP|Flannel services|Manager/Compute|
|8472|UDP|Flannel services|Manager/Compute|
|10256|TCP|kube-proxy health check|Manager + Compute|
|7472|TCP|MetalLB L2 speaker Prometheus metrics|Manager + Compute|
|7946|TCP|MetalLB gossip/memberlist|Manager + Compute|
|2112|TCP|kube-vip Prometheus metrics + health|Manager|
|10257|TCP|Controller manager secure HTTPS port|Manager|
|10259|TCP|Scheduler secure HTTPS port|Manager|
|10249|TCP|kube-proxy Prometheus metrics|Manager + Compute|
|10248|TCP|kubelet local health check|Manager + Compute|
|9099|TCP|Calico Felix health check|Manager + Compute|
|53|TCP/UDP|Kubernetes CoreDNS|Manager|
|443|TCP|NFS StorageClass dynamic provisioner|Compute|
|45845|TCP|CRI-O runtime service|Manager/Compute|
|18515-18520|TCP|DOCA/OFED RDMA and InfiniBand communication port range|Compute|

### Slurm Port Requirements

| Port | Protocol | Service Name | Type of Node |
|---|---|---|---|
|6817|TCP/UDP|slurmctld|Manager (Slurm)|
|6818|TCP/UDP|slurmd|Compute (Slurm)|
|6819|TCP/UDP|slurmdbd|Manager (Slurm)|
|60001-63000|TCP|Slurm SrunPortRange|Manager + Compute|
|3306|TCP|MariaDB|Manager|

### OpenLDAP Port Requirements

| Port Number | Layer 4 Protocol | Purpose | Node |
|-------------|------------------|---------|------|
| 80 | TCP | HTTP | Manager / Login Node |
| 443 | TCP | HTTPS | Manager / Login Node |
| 389 | TCP | LDAP | Manager / Login Node |
| 636 | TCP | LDAPS | Manager / Login Node |

### Telemetry Ports

Telemetry ports are service-cluster traffic requirements. The iDRAC MySQL
service is a headless Kubernetes service and is not exposed as a LoadBalancer,
NodePort, or customer-facing database endpoint. Ports 3306 and 33060 must remain
restricted to the trusted service-cluster network.

| Port | Protocol | Service Name | Type of Node |
|---|---|---|---|
|8161|TCP|ActiveMQ Console|Manager (Telemetry K8s)|
|61613|TCP|ActiveMQ STOMP (port 1)|Manager (Telemetry K8s)|
|61616|TCP|ActiveMQ STOMP (port 2)|Manager (Telemetry K8s)|
|8082|TCP|Telemetry Config UI|Manager (Telemetry K8s)|
|3306|TCP|iDRAC MySQL primary (cluster internal)|Service Kubernetes cluster|
|33060|TCP|iDRAC MySQL X Protocol (cluster internal)|Service Kubernetes cluster|
|9092|TCP|Kafka broker plaintext|Manager (Telemetry K8s)|
|9093|TCP|Kafka broker TLS|Manager (Telemetry K8s)|
|9094|TCP|Kafka LoadBalancer|Manager (Telemetry K8s)|
|8443|TCP|VictoriaMetrics service|Manager (Telemetry K8s)|
|8480|TCP|VictoriaMetrics Insert LB|Manager (Telemetry K8s)|
|8481|TCP|VictoriaMetrics Query LB|Manager (Telemetry K8s)|
|2112|TCP|vmagent self-metrics|Manager (Telemetry K8s)|
|8429|TCP|vmagent remote_write receiver|Manager (Telemetry K8s)|
|9427|TCP|vlagent JSON Lines receiver|Manager (Telemetry K8s)|
|9481|TCP|VictoriaLogs vlinsert|Manager (Telemetry K8s)|
|9491|TCP|VictoriaLogs vlstorage health|Manager (Telemetry K8s)|
|9471|TCP|VictoriaLogs vlselect query|Manager (Telemetry K8s)|
|8687|TCP|vector-ldms health|Manager (Telemetry K8s)|
|9599|TCP|vector-ldms metrics|Manager (Telemetry K8s)|
|8688|TCP|vector-ome health|Manager (Telemetry K8s)|
|9600|TCP|vector-ome metrics|Manager (Telemetry K8s)|
|514|TCP/UDP|Syslog plaintext (VLAgent)|Manager (Telemetry K8s)|
|6514|TCP|Syslog TLS (VLAgent)|Manager (Telemetry K8s)|
|6001-6100|TCP|LDMS aggregator|Manager (Telemetry)|
|6001-6100|TCP|LDMS store daemon|Manager (Telemetry)|
|10001-10100|TCP|LDMS sampler|Compute|

### Build Stream Ports

| Port | Protocol | Service Name | Type of Node |
|---|---|---|---|
|8010|TCP|Build Stream API|Manager (OIM)|

### DOCA/IB Ports

| Port | Protocol | Service Name | Type of Node |
|---|---|---|---|
|18515-18520|TCP/UDP|DOCA/OFED RDMA|Compute (IB nodes)|

### OpenCHAMI Ports

| Port | Protocol | Service Name | Type of Node |
|---|---|---|---|
|8081|TCP|HAProxy HTTP|Manager (OIM)|
|8443|TCP|HAProxy HTTPS|Manager (OIM)|
|27779|TCP|State Mgmt Daemon (SMD)|Manager (OIM)|
|27778|TCP|Boot Script Service (BSS)|Manager (OIM)|
|5432|TCP|PostgreSQL|Manager (OIM)|
|9000|TCP|Step CA (PKI)|Manager (OIM)|
|4444/4445|TCP|Hydra OAuth2|Manager (OIM)|
|67/69|UDP|CoreDHCP|Manager (OIM)|
|53|TCP/UDP|CoreDNS|Manager (OIM)|

## Data Security

Omnia does not store data. The passwords Omnia accepts as input to configure the third party tools are validated and then encrypted using Ansible Vault. Run the following commands routinely on the OIM for the latest RHEL security updates.

```bash
yum update --security
```

For more information on the passwords used by Omnia, see [Login Security Settings](product_subsystem_security.md#login-security-settings)

## Auditing and Logging

!!! note

    All log paths referenced in this section are on the OIM host filesystem.

Omnia writes domain execution logs and service logs on the OIM. The primary
locations are listed below.

**Omnia Log File Locations**

| Location | Purpose |
|----------|---------|
| `/var/log/omnia/discovery/discovery.log` | Discovery playbook log |
| `/var/log/omnia/repo_manager/repo_manager.log` | Repo Manager playbook log |
| `/var/log/omnia/image_build_manager/image_build_manager.log` | Image Build Manager playbook log |
| `/var/log/omnia/orchestrator/orchestrator.log` | Orchestrator playbook log |
| `/var/log/omnia/telemetry/telemetry.log` | Telemetry playbook log |
| `/var/log/omnia/utils/utils.log` | Utils playbook log |
| `<OMNIA_DATA_PATH>/orchestrator/log/openchami/` | OpenCHAMI logs |
| `<OMNIA_DATA_PATH>/repo_manager/log/` | Repository processing and Pulp logs |

Additionally, an aggregate of the events taking place during storage, scheduler and network role installation called `omnia.log` is created in `/var/log`.

There are separate logs generated by the third party tools installed by Omnia.

## Logs

A sample of the omnia.log is provided below:

```bash
2021-02-15 15:17:36,877 p=2778 u=omnia n=ansible | [WARNING]: provided hosts
list is empty, only localhost is available. Note that the implicit localhost does not
match 'all'
2021-02-15 15:17:37,396 p=2778 u=omnia n=ansible | PLAY [Executing omnia roles]
************************************************************************************
2021-02-15 15:17:37,454 p=2778 u=omnia n=ansible | TASK [Gathering Facts]
*****************************************************************************************
*
2021-02-15 15:17:38,856 p=2778 u=omnia n=ansible | ok: [localhost]
2021-02-15 15:17:38,885 p=2778 u=omnia n=ansible | TASK [common : Mount Path]
**************************************************************************************
2021-02-15 15:17:38,969 p=2778 u=omnia n=ansible | ok: [localhost]
...
```

These logs are intended to enable debugging.

!!! note

    Omnia recommends applying masking rules to personally identifiable information (PII) in log files before sending them to external monitoring applications or other third-party destinations.

## Logging Format

Every log message begins with a timestamp and also carries information on the invoking play and task.

The format is described in the following table.

**Log Format Reference**

| Field | Format | Sample Value |
|-------|--------|--------------|
| Timestamp | `yyyy-mm-dd h:m:s` | `2021-02-15 15:17:36` |
| Process ID | `p=xxxx` | `p=2778` |
| User | `u=xxxx` | `u=omnia` |
| Name of the Executing Process | `n=xxxx` | `n=ansible` |
| Task Being Executed | `PLAY` / `TASK` | `PLAY [Executing omnia roles]`<br>`TASK [Gathering Facts]` |
| Error | `fatal: [hostname]: Error Message` | `fatal: [localhost]: FAILED! => {"msg": "lookup_plugin.lines"}` |
| Warning | `[WARNING]: warning message` | `[WARNING]: provided hosts list is empty` |


## Network Vulnerability Scanning

Omnia performs network and application security scans on all modules of the product. Omnia additionally performs Blackduck scans on the open source softwares, which are installed by Omnia at runtime. However, Omnia is not responsible for the third-party software installed using Omnia. Review all third party software before using Omnia to install it.

If you have any feedback about Omnia documentation, please reach out at [omnia.readme@dell.com](mailto:omnia.readme@dell.com).














