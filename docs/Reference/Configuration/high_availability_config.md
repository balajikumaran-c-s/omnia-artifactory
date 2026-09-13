# high_availability_config.yml

This file configures Kubernetes control plane high availability (HA) using a
virtual IP address and load-balanced API servers.

The current provisioning path consumes the first entry in
`service_k8s_cluster_ha`; it does not select an entry by `cluster_name`. It also
uses `virtual_ip_address` to generate kube-vip and Kubernetes API configuration
regardless of the `enable_k8s_ha` value. Keep the intended entry first, set
`enable_k8s_ha: true`, and provide a valid VIP.

## Parameter Reference

--8<-- "html/high_availability_config.html"

## Prerequisites

- Add the matching service Kubernetes cluster to `omnia_config.yml`.
- The `virtual_ip_address` must be a free IP on the admin network subnet --
  it must not be assigned to any physical server or DHCP range.
- Verify the cluster-name relationship, control-plane count, VIP subnet, and
  address conflicts manually. The current input validator does not perform
  these HA-specific cross-checks.

## Usage example

```yaml title="File: $ORCHESTRATOR_DATA_PATH/input/$OMNIA_PROJECT_NAME/high_availability_config.yml"
---
service_k8s_cluster_ha:
  - cluster_name: service_cluster
    enable_k8s_ha: true
    virtual_ip_address: "172.16.107.1"
```

!!! info

    - [Omnia Config](omnia_config.md) -- Kubernetes deployment settings.
    - [Minimum Nodes](../ClusterRequirements/minimum_nodes.md) -- Minimum node counts for HA deployments.
    - [Ports](../../SecurityConfigurationGuide/network_security.md#kubernetes-port-requirements) -- Kubernetes ports including
      the API server.














