
# network_spec.yml

This file defines the admin network, the optional InfiniBand network, and any
additional DHCP subnets. Orchestrator and Discovery stage independent copies
of this file in their respective project input directories.

## Location

```text
$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/network_spec.yml
$OMNIA_DATA_PATH/discovery/input/$OMNIA_PROJECT_NAME/network_spec.yml
```

The default paths are under `/opt/omnia`. Configure the Discovery copy for
node discovery and the Orchestrator copy for provisioning.

Discovery currently reads only `admin_network.subnet` and
`ib_network.subnet`. It uses their first two octets with the last two octets of
each server's BMC address to derive `ADMIN_IP` and `IB_IP`. The Discovery
`validate` tag does not validate this file. Orchestrator consumes and validates
the remaining network fields from its own copy.

## Top-level structure

`network_spec.yml` contains a single top-level key, `Networks`, which is a
YAML list of network definitions.

```yaml title="File: /opt/omnia/orchestrator/input/project_default/network_spec.yml"
Networks:
  - admin_network:
      additional_subnets: []
  - ib_network: {}
```
## Admin Network Configuration Parameters
--8<-- "html/network_spec-admin_network.html"

## Infiniband Network Configuration Parameters (optional)
--8<-- "html/network_spec-ib_network.html"

## Additional Subnets Configuration Parameters (optional)
--8<-- "html/network_spec-additional_subnets.html"


## Usage example

```yaml title="File: /opt/omnia/orchestrator/input/project_default/network_spec.yml"
---
Networks:
  - admin_network:
      oim_nic_name: "eno1"
      subnet: "172.16.107.0"
      netmask_bits: "24"
      primary_oim_admin_ip: "172.16.107.254"
      primary_oim_bmc_ip: ""
      router: "172.16.107.254"
      dynamic_range: "172.16.107.201-172.16.107.250"
      dns: []
      ntp_servers: []
      additional_subnets:
        - subnet: "10.40.1.0"
          netmask_bits: "24"
          router: "10.40.1.1"
          dynamic_range: "10.40.1.100-10.40.1.200"
        - subnet: "10.40.3.0"
          netmask_bits: "24"
          router: "10.40.3.1"
          dynamic_range: "10.40.3.100-10.40.3.200"

  - ib_network:
      subnet: "192.168.0.0"
      netmask_bits: "24"
      dns: []
```


!!! note

    - The `router` field is optional and specifies the gateway IP address for the admin network. The configured value is advertised to nodes via DHCP as their default gateway.
    - In connected deployments, set `router` to the external rack gateway (such as a SONiC switch) which provides connectivity beyond the rack network.
    - In air-gapped deployments, set `router` to the OIM's IP address if the OIM is acting as the rack gateway. If a dedicated router or gateway is available, specify its IP address instead.
    - Default value: `172.16.107.254`
    - The `dynamic_range` must not overlap with any static IPs assigned
      in the PXE mapping file.

!!! info

    - [Network Topologies](../SupportMatrix/network_topologies.md) -- How topologies
      affect NIC and VLAN assignments.
    - [Nics](../SupportMatrix/nics.md) -- Supported NIC models.














