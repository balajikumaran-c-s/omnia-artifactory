
# omnia_config.yml

This file controls the deployment of Slurm and Kubernetes across cluster nodes.

## Parameter reference
### Slurm Configuration Parameters

--8<-- "html/omnia_config-slurm_cluster.html"

This release supports one Slurm cluster configuration. When Slurm is enabled,
supply one item in `slurm_cluster`; the current implementation reads only the
first item and does not process additional items. `vast_storage_name` is
optional. When it is empty or omitted, Orchestrator reuses `nfs_storage_name`
for Slurm shared-data and HPC-tools paths and skips the standard mount whose
`name` is `vast_storage`. When set, `vast_storage_name` must exactly match one
mount `name` in `storage_config.yml`.

### Kubernetes Configuration Parameters

--8<-- "html/omnia_config-k8s_cluster.html"

PowerScale CSI is controlled by `enable_powerscale_csi` on the
`service_k8s_cluster` selected with `deployment: true`. The optional Boolean
defaults to `false`. When set to `true`, both
`csi_powerscale_driver_secret_file_path` and
`csi_powerscale_driver_values_file_path` are required and must identify
existing regular files by absolute path. Catalog membership and populated file
paths do not enable CSI when the flag is `false` or omitted.

When service Kubernetes is configured, set `deployment: true` on exactly one
`service_k8s_cluster` item. Other entries may remain in the list with
`deployment: false`, but Orchestrator deploys only the selected item. The
current runtime falls back to the first list item when none is marked; use an
explicit selection so that list reordering cannot change the deployed cluster.

## Usage example

```yaml title="File: $ORCHESTRATOR_DATA_PATH/input/$OMNIA_PROJECT_NAME/omnia_config.yml"
---
slurm_cluster:
  - cluster_name: slurm_cluster
    nfs_storage_name: nfs_slurm
    vast_storage_name: vast_storage
    node_discovery_mode: "homogeneous"
    # Optional: Override Slurm and cgroup configuration
    config_sources:
      slurm: /path/to/custom/slurm.conf
      cgroup: /path/to/custom/cgroup.conf
      # slurm:
      #   SlurmctldTimeout: 60
      #   SlurmdTimeout: 150
    # Optional: Override hardware specs for specific node groups
    node_hardware_defaults:
      grp1:
        sockets: 2
        cores_per_socket: 64
        threads_per_core: 2
        real_memory: 512000
        gres: "gpu:4"
      grp2:
        sockets: 2
        cores_per_socket: 32
        threads_per_core: 2
        real_memory: 256000

service_k8s_cluster:
  - cluster_name: service_cluster
    deployment: true
    enable_powerscale_csi: false
    etcd_on_local_disk: false
    k8s_cni: "calico"
    pod_external_ip_range: "172.16.107.170-172.16.107.200"
    k8s_service_addresses: "10.233.0.0/18"
    k8s_pod_network_cidr: "10.233.64.0/18"
    nfs_storage_name: "nfs_k8s"
    k8s_crio_storage_size: "20G"
    csi_powerscale_driver_secret_file_path: ""
    csi_powerscale_driver_values_file_path: ""
```


!!! info

    - [Orchestrator Config](orchestrator_config.md) -- Provisioning, catalog path, and upstream output settings.
    - [Slurm Conf](../SampleFiles/slurm_conf.md) -- Custom Slurm configuration.
    - [HA Config](high_availability_config.md) -- Kubernetes high-availability settings.
    - [Slurm Storage Architecture](../../HowTo/orchestrator/deploy_slurm.md#slurm-storage-architecture) -- How NFS and VAST mounts are used by Slurm.
    - [K8s Storage Architecture](../../HowTo/orchestrator/deploy_kubernetes.md#k8s-storage-architecture) -- How NFS mounts are used by service K8s.












