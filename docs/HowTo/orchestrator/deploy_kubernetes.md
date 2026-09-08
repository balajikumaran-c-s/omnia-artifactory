# Deploy Service Kubernetes

## Overview

Orchestrator prepares service Kubernetes nodes whose functional-group names
start with `service_kube_`. It registers those nodes and groups in OpenCHAMI,
creates their boot and cloud-init data, mounts the selected storage on the OIM,
stages Kubernetes configuration and offline content on that storage, and
configures OpenLDAP clients when the catalog enables OpenLDAP.

The source provides x86_64 templates for the first control-plane node,
additional control-plane nodes, and worker nodes. During functional-group
generation, the first `service_kube_control_plane_x86_64` group is converted to
`service_kube_control_plane_first_x86_64`.

## Prerequisites

- Complete Repo Manager and Image Build Manager with Kubernetes content and an
  image for each `service_kube_` functional group.
- Add the Kubernetes nodes to the PXE mapping with their service tags,
  lowercase hostnames, admin network data, and BMC data for physical nodes.
- Configure the OIM admin network and any additional node subnets in
  `network_spec.yml`. The Kubernetes role derives the node-network CIDRs from
  the mapped control-plane and worker admin IPs.
- Provide an NFS mount whose `name` matches `nfs_storage_name` in
  `omnia_config.yml`. The OIM must be able to mount it and create the Kubernetes
  configuration directories.
- Configure `high_availability_config.yml`; its `cluster_name` must match the
  Kubernetes entry selected for deployment.
- If the catalog enables the PowerScale CSI driver, provide the secret and
  values file paths requested by `omnia_config.yml`.

### K8s storage architecture

The mount selected by `nfs_storage_name` is used to stage the Kubernetes
configuration, SSH key, package layout, Pulp certificate, and offline Calico,
MetalLB, Helm, and NFS provisioner content. The role also creates per-node
directories for Kubernetes, kubelet, pod logs, and, for control-plane nodes,
etcd data.

## Procedure

1. Assign nodes to the supported Kubernetes functional groups in
   `pxe_mapping_file.csv`:

    ```text title="pxe_mapping_file.csv — functional-group examples"
    service_kube_control_plane_x86_64
    service_kube_node_x86_64
    ```

   The source classification accepts the `service_kube_` prefix. When the
   canonical control-plane name shown above is used, functional-group
   generation marks the first occurrence as
   `service_kube_control_plane_first_x86_64`.

2. Configure the Kubernetes cluster in `omnia_config.yml`. Exactly one source
   template entry is marked `deployment: true`.

    ```yaml title="omnia_config.yml"
    service_k8s_cluster:
      - cluster_name: service_cluster
        deployment: true
        etcd_on_local_disk: false
        k8s_cni: "calico"
        pod_external_ip_range: "<external-ip-range-or-cidr>"
        k8s_service_addresses: "10.233.0.0/18"
        k8s_pod_network_cidr: "10.233.64.0/18"
        nfs_storage_name: "nfs_k8s"
        k8s_crio_storage_size: "20G"
        csi_powerscale_driver_secret_file_path: ""
        csi_powerscale_driver_values_file_path: ""
    ```

   The source input supports `calico` or `flannel`; it directs RoCE deployments
   to use `flannel`. Keep the external IP range unused by cluster nodes and
   keep the service and pod networks unused in the surrounding infrastructure.

3. Configure the matching HA entry:

    ```yaml title="high_availability_config.yml"
    service_k8s_cluster_ha:
      - cluster_name: service_cluster
        enable_k8s_ha: true
        virtual_ip_address: "<unused-admin-network-ip>"
    ```

4. Configure the NFS mount. Replace the source placeholder with a reachable
   export.

    ```yaml title="storage_config.yml"
    mounts:
      - name: "nfs_k8s"
        source: "<nfs-server>:<export>"
        mount_point: "/opt/omnia/k8s_mount"
        fs_type: "nfs"
        mnt_opts: "nosuid,rw,sync,hard,intr"
        mount_on_oim: true
        functional_group_prefix: ["service_kube"]
    ```

5. Validate, deploy the OIM services, and provision. The `provision` tag also
   processes any Slurm, OS-only, login, or custom groups in the same mapping.

    ```bash title="Run on: OIM"
    cd /omnia/src/orchestrator
    ansible-playbook playbooks/orchestrator.yml --tags validate
    ansible-playbook playbooks/orchestrator.yml --tags precheck
    ansible-playbook playbooks/orchestrator.yml --tags prepare
    ansible-playbook playbooks/orchestrator.yml --tags provision
    ```

6. For physical servers, start the PXE and node-registration flow:

    ```bash title="Run on: OIM"
    ansible-playbook playbooks/orchestrator.yml --tags pxeboot
    ```

## Verification

Confirm the Orchestrator view first:

```bash title="Run on: OIM"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/provisioning_report.yml"
cat "$OMNIA_DATA_PATH/orchestrator/output/$OMNIA_PROJECT_NAME/orchestrator_status.yml"
```

After cloud-init completes, use the commands embedded in the source templates
on the first control-plane node:

```bash title="Run on: first Kubernetes control-plane node"
kubectl get nodes -o wide
kubectl get pods --all-namespaces -o wide
```

All mapped nodes should appear in the first command. Workloads in the second
command should reach `Running` or `Completed`.

## Next steps

- Use [Add Nodes](../../Operations/add_nodes.md) for additional control-plane or worker entries
  that have matching source templates and Image Build Manager artifacts.
- Use [Configure HA](configure_kubernetes_ha.md) and
  [Configure Storage](configure_storage.md) for the associated
  project inputs.
- If selected by the catalog, configure the PowerScale CSI files before
  rerunning provisioning.

## Troubleshooting

**The NFS configuration directory cannot be created**

Confirm that the configured server exports the selected path with permissions
that allow the OIM to write it. The role's source guidance uses an export with
`rw,sync,no_root_squash,no_subtree_check`; reload exports and restart the NFS
server after correcting it.

**Offline Kubernetes variables are missing**

Confirm that `repo_status.yml` provides `offline_tarball_path` and
`offline_manifest_path`, its Pulp certificate exists, and the catalog contains
the Kubernetes package definitions selected by the role. Run Repo Manager
again before retrying Orchestrator.

**A Kubernetes node is not ready**

Inspect the commands used in the generated cloud-init workflow:

```bash title="Run on: affected Kubernetes node"
systemctl status kubelet
```

Then inspect the cluster from the first control-plane node:

```bash title="Run on: first Kubernetes control-plane node"
kubectl get nodes -o wide
kubectl get pods --all-namespaces -o wide
```

Also check the node's cloud-init output and the corresponding entry in
`orchestrator_status.yml` before rerunning the relevant phase.
