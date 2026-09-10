# high_availability_config.yml

This file configures Kubernetes control plane high availability (HA) using a
virtual IP address and load-balanced API servers.

## Parameter Reference

--8<-- "html/high_availability_config.html"

## Prerequisites

- Add the matching service Kubernetes cluster to `omnia_config.yml`.
- The `virtual_ip_address` must be a free IP on the admin network subnet --
  it must not be assigned to any physical server or DHCP range.

## Usage example

```yaml title="File: /opt/omnia/orchestrator/input/project_default/high_availability_config.yml"
---
service_k8s_cluster_ha:
  - cluster_name: service_cluster
    enable_k8s_ha: true
    virtual_ip_address: "172.16.107.1"
```
!!! info

    - [Omnia Config](omnia_config.md) -- Kubernetes deployment settings.
    - [Minimum Nodes](../../Reference/../Reference/ClusterRequirements/minimum_nodes.md) -- Minimum node counts for HA deployments.
    - [Ports](../../SecurityConfigurationGuide/network_security.md#kubernetes-port-requirements) -- Kubernetes ports including
      the API server.
















