# Discover nodes using OME

## Overview

The Omnia Discovery module connects to Dell OpenManage Enterprise (OME) from
the Omnia Infrastructure Manager (OIM), collects server and network-interface
inventory, and generates the node mapping used by the Orchestrator module.
Discovery runs locally on the OIM and supports OME as its discovery mechanism.

A complete untagged run:

1. Validates `discovery_config.yml`.
2. Creates or loads the OME credentials stored for the selected project.
3. Verifies that OME is reachable on TCP port 443.
4. Collects OME devices of server type `1000`, their iDRAC details, Ethernet
   interfaces, InfiniBand interfaces, and OME static-group membership.
5. Generates a timestamped PXE mapping file and a NIC-status report.
6. Once the OME discovery role starts, writes `discovery_status.yml` with the
   result of that role.

With the default data path and project, the generated files are under
`/opt/omnia/discovery/output/project_default/`:

| File | Purpose |
|------|---------|
| `bmc_pxe_mapping_file_<timestamp>.csv` | Timestamped node mapping for review and downstream use. |
| `bmc_pxe_mapping_file.csv` | Symbolic link to the latest timestamped mapping file. |
| `bmc_discovery_report_<timestamp>.csv` | Point-in-time report of BMC, Ethernet, and InfiniBand NIC status. |
| `discovery_status.yml` | OME role status, mapping-file path, discovered-server count, and timestamp. This file is not updated by earlier failures. |

Discovery generates the mapping required by the downstream deployment
workflow. It does not build OS images.

## Prerequisites

- Run Discovery on the OIM with permission to create files under the Omnia data
  path, `/var/log/omnia/discovery/`, and `/opt/omnia/log/core/playbooks/`.
- Complete [OIM setup](../main/setup_oim.md). The main setup runs the Discovery
  initialization script, installs its dependencies, creates its runtime and log
  directories, and stages its input templates.
- If Discovery was skipped during main setup, initialize it with
  `./omnia.sh --init discovery` from `src/main` before the first run.
- Use the module-initialized environment. `requirements.txt` requests Ansible
  Core 2.20 or later, Jinja 3.0 or later, and PyYAML 6.0.3 or later;
  `requirements.yml` requests `ansible.posix` 2.0.0 and `community.general`
  10.3.0. The `requests` Python library must also be importable by Ansible, and
  `openssl` must be available to generate a Vault password when the credential
  files do not exist.
- OME must be reachable from the OIM over TCP port 443.
- Valid OME credentials must be available. Discovery prompts for missing OME
  credentials, saves them in `discovery_credentials.yml`, and encrypts the file
  with Ansible Vault using `.discovery_credentials_key`.
- Target servers must already be managed by OME as server devices and must have
  a service tag. Devices without a service tag are not added to the discovered
  server list.
- OME must expose the device, group, device-management, and server network
  interface inventory used by Discovery.

### Input contract

Discovery resolves its runtime paths from the following values:

| Input | Required value or default |
|-------|---------------------------|
| `OMNIA_DATA_PATH` | Optional. Defaults to `/opt/omnia`. |
| `OMNIA_PROJECT_NAME` | Optional. Defaults to `project_default`. |
| `project_name` | Optional Ansible extra variable used when `OMNIA_PROJECT_NAME` is not set. |

Prepare the following files in
`<OMNIA_DATA_PATH>/discovery/input/<OMNIA_PROJECT_NAME>/`:

| Input | Requirement |
|-------|-------------|
| `discovery_config.yml` | Required. Contains `enable_bmc_discovery` and the OME IPv4 address in `ome_ip`. |
| `network_spec.yml` | Required during execution. Discovery reads the admin and InfiniBand subnet values from this file to derive node IP addresses. |
| `discovery_credentials.yml` | Created automatically if absent. Contains `ome_username` and `ome_password` and is stored encrypted. |
| `.discovery_credentials_key` | Created automatically with the credential file and stored with mode `0400`. |

`discovery_config.yml` must retain both schema fields. For an OME discovery
run, set `enable_bmc_discovery: true` and set `ome_ip` to a valid,
non-loopback IPv4 address.

The credential workflow requires a nonempty OME username and password. The
username must contain between 1 and 64 characters and cannot contain a
backslash, single quote, double quote, or semicolon. The password must contain
between 1 and 128 characters. Discovery prompts only for values that are empty
or absent in the credential file.

### Plan iDRAC hostnames

Discovery uses the iDRAC hostname reported by OME to derive the physical
`GROUP_NAME` written to the PXE mapping file. Configure consistent iDRAC
hostnames before running Discovery so that servers in the same Scalable Unit
resolve to the same group.

Use the following complete naming convention when encoding the server's
physical location:

```text
idrac-<SU><1-100>R<000-999>OU<1-54><Type><Instance>
```

| Component | Description | Recommended format |
|-----------|-------------|--------------------|
| `SU` | Scalable Unit containing the server. | `SU1` through `SU100`; matching is case-insensitive. |
| `R` | Rack within the Scalable Unit. | `R1` through `R999`. |
| `OU` | Open Rack v3 unit position within the rack. | `OU1` through `OU54`. |
| `Type` | Server type at the rack position. | Use `C` for a compute node. |
| `Instance` | Individual server instance at the rack position. | `1` through `99`. |

For example:

```text title="Example breakdown"
SU02   R1   OU05   C7
│      │     │      │
│      │     │      └── Compute node instance
│      │     └───────── Open Rack v3 unit position
│      └─────────────── Rack within the Scalable Unit
└────────────────────── Scalable Unit
```

`idrac-SU02R1OU05C7` identifies compute node 7 at unit position 5 in rack 1
of Scalable Unit 02.

The current mapping generator searches the OME-reported hostname for a
case-insensitive `SU[optional-letter]<digits>R<digits>` sequence and writes the
matched `SU` portion in uppercase. The complete naming convention and numeric
ranges above are operational planning requirements; Discovery does not
validate the entire hostname or those ranges.

| OME-reported iDRAC hostname | Generated `GROUP_NAME` |
|-----------------------------|------------------------|
| `idrac-SU02R1OU05C7` | `SU02` |
| `idrac-SUA99R999OU30C2` | `SUA99` |
| `SU1R2OU1C5` | `SU1` |
| `idrac-JCGT033` | `grp0` |

!!! warning

    OME can report an instrumentation name, a DNS name, or its device name for
    the iDRAC. Verify the value visible in OME before running Discovery. If the
    reported hostname does not contain a recognized `SU...R...` sequence,
    Discovery uses `grp0`. An incorrect `GROUP_NAME` can also prevent or
    misdirect `PARENT_SERVICE_TAG` assignment for Slurm compute nodes.

### Plan OME static groups

When OME exposes a `Static Groups` container, Discovery uses its immediate
child groups as functional-group assignments. If that container is absent,
Discovery falls back to non-system OME groups. A server can belong to no more
than one of the groups that Discovery processes; Discovery stops if a server
belongs to multiple processed groups.

The mapping generator accepts these exact, case-sensitive OME static-group
names:

- `service_kube_control_plane_x86_64`
- `service_kube_node_x86_64`
- `login_node_x86_64`
- `login_node_aarch64`
- `login_compiler_node_x86_64`
- `login_compiler_node_aarch64`
- `slurm_control_node_x86_64`
- `slurm_node_x86_64`
- `slurm_node_aarch64`
- `os_x86_64`
- `os_aarch64`

#### Create OME static groups

Create one static group for each Omnia functional group required by the
cluster:

1. In the OME left navigation menu, select **CUSTOM GROUPS > Static Groups**.
2. Select the ellipsis (**...**) next to **Static Groups**, and then select
   **Create Group**.
3. Enter one of the supported functional-group names exactly as listed above.
4. Enter a description that identifies the purpose of the group.
5. Select **Finish**.

Repeat these steps for each functional group required by the cluster. The
group name determines the value written to `FUNCTIONAL_GROUP_NAME`; the group
description does not affect Discovery behavior.

#### Assign devices to OME static groups

After creating the required static groups, assign the discovered servers:

1. Select the static group from the OME group list.
2. Select **Add Devices**.
3. In the **Add Devices to Group** dialog box, select the servers whose
   intended role matches the functional group.
4. Select **Finish**.

Repeat these steps for the remaining groups. Before running Discovery, verify
that each server is assigned to no more than one Omnia static group and that
each group is an immediate child of **Static Groups**.

A server without a static-group assignment is placed in
`slurm_node_aarch64`. A server assigned to a nonempty, unsupported static
group is skipped when the mapping file is generated, although it remains in
the discovery report.

For `slurm_node_x86_64` and `slurm_node_aarch64`, Discovery populates
`PARENT_SERVICE_TAG` from a `service_kube_node_x86_64` server with the same
derived `GROUP_NAME`. Plan the iDRAC hostnames and static-group membership
accordingly when this relationship is required.

## Procedure

1. In OME, discover or manage the target servers that Omnia will provision.
   Confirm that each target appears in OME as a server device and has a service
   tag. Omnia queries the existing OME inventory; it does not add devices to
   OME. For the version-specific device-discovery procedure, see the
   [Dell OpenManage Enterprise documentation](https://www.dell.com/support/product-details/en-us/product/dell-openmanage-enterprise/docs){target="_blank"}.

2. [Configure the Main environment](../main/configure_environment.md). The
   examples on this page use `OMNIA_DATA_PATH=/opt/omnia` and
   `OMNIA_PROJECT_NAME=project_default`.

3. Complete OIM setup. If Discovery was skipped during setup, initialize only
   this domain:

    ```bash title="Run on: OIM host"
    cd src/main
    ./omnia.sh --init discovery
    ```

    Initialization installs the declared dependencies, creates the runtime and
    log directories, and copies `discovery_config.yml` and `network_spec.yml` to
    `<OMNIA_DATA_PATH>/discovery/input/<OMNIA_PROJECT_NAME>/`. Review its output
    and confirm that the dependencies are available.

    !!! warning

        If the destination already contains files, initialization asks before
        overwriting them. Preserve any project-specific changes when responding
        to the prompt.

4. [Create the required OME static groups](#create-ome-static-groups), and then
   [assign the discovered servers](#assign-devices-to-ome-static-groups). Use
   an exact supported functional-group name and place each server in no more
   than one Omnia static group. A server without a static-group assignment
   uses the default functional group.

5. Edit the staged `discovery_config.yml` and enable OME discovery:

    ```yaml title="File: /opt/omnia/discovery/input/project_default/discovery_config.yml"
    enable_bmc_discovery: true
    ome_ip: "192.168.1.100"
    ```

    Configure `ome_ip` with a valid, non-loopback OME IPv4 address, which is the
    format defined by the Discovery configuration schema.

6. Edit the staged `network_spec.yml`. Discovery uses only
   `admin_network.subnet` and `ib_network.subnet`; preserve the `Networks` list
   and both network entries from the staged template.

    ```yaml title="File: /opt/omnia/discovery/input/project_default/network_spec.yml"
    Networks:
      - admin_network:
          subnet: "172.16.0.0"

      - ib_network:
          subnet: "192.168.0.0"
    ```

    Discovery derives `ADMIN_IP` by combining the first two octets of the admin
    subnet with the last two octets of the server's BMC IP. It derives `IB_IP`
    in the same way from the InfiniBand subnet, but only when an InfiniBand NIC
    was detected. The Discovery validator does not validate `network_spec.yml`,
    so review these subnet values before execution.

7. From `src/main`, validate `discovery_config.yml` before contacting OME:

    ```bash title="Run on: OIM host"
    ./omnia.sh --run discovery --tags validate
    ```

    A successful validation prints `Discovery configuration validation passed.`
    and displays the validation-log path. This phase does not request OME
    credentials.

8. Run the complete Discovery workflow:

    ```bash title="Run on: OIM host"
    ./omnia.sh --run discovery
    ```

    When `discovery_credentials.yml` does not exist, Discovery creates it and
    creates its Vault key if needed. Enter the OME username and password when
    prompted. The password prompt asks for confirmation before the credential
    file is encrypted.

    The default untagged run performs setup, validation, credential handling,
    and OME discovery. The `execute` and `discovery` tags route to the same OME
    execution flow. The `credentials` tag updates credentials without running
    discovery. Use only one tag in a command. See [Run
    Discovery](index.md#run-discovery) for the complete tag table, including
    the lifecycle placeholders that do not perform work in this release.

    A successful run without BuildStream prints a completion summary in this
    form:

    ```text title="Expected output"
    ============================================================
    OME Discovery Complete
    ============================================================
    BMC PXE mapping file generated: /opt/omnia/discovery/output/project_default/bmc_pxe_mapping_file_<timestamp>.csv
    BMC discovery report generated: /opt/omnia/discovery/output/project_default/bmc_discovery_report_<timestamp>.csv
      (Lists link status of BMC, Ethernet, and InfiniBand NICs for each server)
    Total servers discovered: <count>

    Output directory: /opt/omnia/discovery/output/project_default

    Next Steps:
    1. Review and edit the generated PXE mapping file.
    2. Review the discovery report for NIC link statuses.
    3. Update HOSTNAME, FUNCTIONAL_GROUP_NAME, GROUP_NAME as needed.
    4. Copy the mapping file to the Orchestrator input directory.
    ============================================================
    ```

    The current Discovery implementation may print a Build Stream-specific
    completion message only when a `build_stream_config.yml` is present in the
    Discovery input directory. That file is not part of the Discovery input
    contract. Follow the Build Stream handoff in [Next steps](#next-steps)
    instead of copying another domain's configuration into this directory.

## Verification

1. **Output contract:** After a successful OME discovery, confirm that the
   project output directory,
   `<OMNIA_DATA_PATH>/discovery/output/<OMNIA_PROJECT_NAME>/`, contains the
   following artifacts:

    | Artifact | Verification |
    |----------|--------------|
    | `bmc_pxe_mapping_file_<timestamp>.csv` | Timestamped PXE mapping containing discovered servers whose OME static-group assignment is supported or empty. |
    | `bmc_pxe_mapping_file.csv` | Symbolic link to the latest timestamped mapping file. |
    | `bmc_discovery_report_<timestamp>.csv` | NIC-status report generated from every discovered server that has a service tag. |
    | `discovery_status.yml` | Status of the OME discovery role, mapping-file path, discovered-server count, and timestamp. |

    With the standard data path and project name, check the status file:

    ```bash title="Run on: OIM host"
    cat /opt/omnia/discovery/output/project_default/discovery_status.yml
    ```

    A successful status file has this structure:

    ```yaml title="Expected structure"
    overall_status: "success"
    discovery_mechanism: "ome"
    bmc_pxe_mapping_file: "/opt/omnia/discovery/output/project_default/bmc_pxe_mapping_file_<timestamp>.csv"
    servers_discovered: <count>
    timestamp: "<ISO 8601 timestamp>"
    ```

    Confirm that `overall_status` is `success`, `discovery_mechanism` is `ome`,
    `bmc_pxe_mapping_file` identifies the timestamped mapping file, and
    `servers_discovered` matches the OME servers expected in the discovery
    report. Setup, validation, and credential failures occur before this status
    file is updated; for those failures, use the Ansible output and log instead
    of relying on an existing status file.

2. Confirm that the stable mapping-file link resolves to the latest
   timestamped file:

    ```bash title="Run on: OIM host"
    readlink /opt/omnia/discovery/output/project_default/bmc_pxe_mapping_file.csv
    ```

3. Review the generated mapping:

    ```bash title="Run on: OIM host"
    cat /opt/omnia/discovery/output/project_default/bmc_pxe_mapping_file.csv
    ```

    The mapping contains these columns:

    | Column | Generated value |
    |--------|-----------------|
    | `FUNCTIONAL_GROUP_NAME` | Supported OME static-group name, or `slurm_node_aarch64` when the server has no assignment. |
    | `GROUP_NAME` | `SU` identifier derived from the iDRAC hostname, or `grp0` when no identifier is found. |
    | `SERVICE_TAG` | Service tag reported by OME. |
    | `PARENT_SERVICE_TAG` | Service tag of a `service_kube_node_x86_64` in the same group for Slurm compute-node roles; otherwise empty. |
    | `HOSTNAME` | `nid` plus a three-digit sequence number based on discovery order. The supported range is `nid000` through `nid999`; automatic generation normally begins with `nid001`, and skipped devices can create gaps. |
    | `ADMIN_MAC` | MAC of the first non-iDRAC, non-InfiniBand port with link status `Up`; otherwise the first usable non-iDRAC, non-InfiniBand port. |
    | `ADMIN_IP` | Admin subnet's first two octets combined with the BMC IP's last two octets. |
    | `BMC_MAC` | iDRAC MAC address reported by OME. |
    | `BMC_IP` | iDRAC IP address reported by OME. |
    | `IB_NIC_NAME` | InfiniBand port identifier selected with priority `Up`, then `Unknown`, then another reported state; empty when no InfiniBand NIC is found. |
    | `IB_IP` | InfiniBand subnet's first two octets combined with the BMC IP's last two octets; empty when no InfiniBand NIC is found. |

4. Review the report with the same timestamp as the mapping file. Replace
   `<timestamp>` with the value in the mapping filename:

    ```bash title="Run on: OIM host"
    cat /opt/omnia/discovery/output/project_default/bmc_discovery_report_<timestamp>.csv
    ```

    The report contains `SERVICE_TAG`, `BMC_MAC`, `BMC_IP`, `BMC_NIC_STATUS`,
    `ETHERNET_NIC_MAC`, `ETHERNET_NIC_LINK_STATUS`, `IB_NIC_NAME`, and
    `IB_NIC_LINK_STATUS`. Use it to identify missing NIC inventory and links
    reported as `Down` or `Unknown` before using the mapping downstream.

    The report includes every discovered server with a service tag. The mapping
    can contain fewer rows when a server was assigned to an unsupported OME
    static group.

## Next steps

1. Review and, where necessary, edit `HOSTNAME`, `FUNCTIONAL_GROUP_NAME`, and
   `GROUP_NAME` in the timestamped mapping file. Also confirm the generated
   service tags, parent relationships, MAC addresses, and IP addresses.

2. Without BuildStream, copy the reviewed mapping to the Orchestrator input
   directory:

    ```bash title="Run on: OIM host"
    cp /opt/omnia/discovery/output/project_default/bmc_pxe_mapping_file.csv \
      /opt/omnia/orchestrator/input/project_default/pxe_mapping_file.csv
    ```

3. With BuildStream enabled, build the images through the build pipeline first.
   If the GitLab server is not yet available, place the reviewed mapping at the
   Orchestrator input path shown above. If GitLab is available, copy it to
   `input/orchestrator/pxe_mapping_file.csv` in the GitLab project and commit
   the change; the commit triggers the deploy pipeline for the nodes in that
   mapping.

## Troubleshooting

### Discovery configuration validation fails

- Confirm that
  `<OMNIA_DATA_PATH>/discovery/input/<project>/discovery_config.yml` exists and
  contains both `enable_bmc_discovery` and `ome_ip`.
- When BMC discovery is enabled, use a non-loopback OME IPv4 address.
- Correct YAML parsing errors reported by the playbook.
- Review
  `/opt/omnia/log/core/playbooks/discovery_validation_<project>.log`, then rerun
  the `validate` tag.

### OME is unreachable

Discovery waits up to 30 seconds for `<ome_ip>:443`. Confirm that `ome_ip` is
correct, OME is powered on, and the OIM can reach TCP port 443.

### OME authentication fails

Correct the `ome_username` and `ome_password` values managed in
`discovery_credentials.yml` before running Discovery again. Rerunning alone
does not replace nonempty stored credentials. If the credential file is
Vault-encrypted, its matching `.discovery_credentials_key` must be present in
the same project input directory.

### No servers are discovered

Confirm that OME manages the target devices as server type `1000` and that the
devices have nonempty service tags. Discovery fails when the filtered server
list is empty.

### A server belongs to multiple OME static groups

Discovery reports each conflicting service tag and its groups, then stops.
Remove the duplicate static-group memberships so that each server belongs to
no more than one static group and rerun Discovery.

### A server is missing from the mapping

Look for a warning that names an unsupported OME static group. Rename the group
to one of the supported functional-group names or remove the server's static
group assignment to use the default. The server remains visible in the
discovery report if OME supplied its service tag.

### The admin MAC address is unexpected or empty

Check `ETHERNET_NIC_MAC` and `ETHERNET_NIC_LINK_STATUS` in the discovery report
and inspect the server network-interface inventory in OME. Discovery excludes
iDRAC and InfiniBand interfaces, selects the first usable port reported as
`Up`, and otherwise falls back to the first usable non-iDRAC,
non-InfiniBand port. If that inventory produces no MAC address, Discovery
attempts the OME `deviceNics` inventory as a fallback.

### InfiniBand fields are empty

This is expected when OME does not report an interface whose identifier
contains `InfiniBand`. When an interface is present, Discovery prefers an `Up`
port, then `Unknown`, and then another reported state.

### Group names or parent service tags are incorrect

Ensure the iDRAC hostname contains an `SU` identifier immediately followed by
an `R` and rack number, such as `SU1R2OU1C5`. For Slurm compute-node roles,
ensure a `service_kube_node_x86_64` server resolves to the same `GROUP_NAME`.

### OME discovery execution fails

If OME execution started, inspect `discovery_status.yml`; a failed OME phase
records `failed_task` and `failure_reason`. Setup, validation, or credential
failures can leave that file absent or unchanged from a previous run. For all
phases, inspect the Ansible output and
`/var/log/omnia/discovery/discovery.log`.

### Discovery is blocked by an upgrade lock

Complete the Omnia upgrade before rerunning Discovery. The workflow does not
run normally while `/opt/omnia/.data/upgrade_in_progress.lock` exists.
