# Configure Additional Cloud-Init

Omnia allows administrators to inject custom cloud-init directives into the node provisioning pipeline. This enables boot-time customization such as writing configuration files or running setup commands on cluster nodes without modifying platform-managed templates.

## Overview

The additional cloud-init feature supports two scopes:

- **common**: Directives applied to all provisioned nodes.
- **groups**: Directives applied only to nodes belonging to a specific functional group as defined in the PXE mapping file.

Both scopes support the following cloud-init directives:

| Directive | Description |
|-----------|-------------|
| `write_files` | Create or append files on the node at boot time. Each entry must include a `path`. |
| `runcmd` | Run shell commands during the final stage of cloud-init. Each entry must be a string. |

!!! caution

    Do not use `bootcmd`, `network`, `network-config`, or `packages`. These
    keys are platform-managed by Omnia and are outside the supported
    additional cloud-init format. The current input validator does not reject
    every unsupported key before provisioning.

## Prerequisites

- The [Discover Nodes](../discovery/discover_nodes.md) procedure is complete,
  and its PXE mapping file is available in the Orchestrator input directory.
- The Orchestrator credential workflow is complete.
- The [Prepare OIM](../main/setup_oim.md) procedure is complete (OpenCHAMI and DHCP are running).
- The [Create Local Repos](../repo_manager/configure_repos.md) procedure is complete (local Pulp repository is set up with required packages).
- The [Build Cluster Images](../image_build_manager/build_images.md) procedure is complete.

## Procedure

### Step 1: Enable additional cloud-init

Edit the Orchestrator configuration file:

```bash title="Run on: OIM"
vi /opt/omnia/orchestrator/input/project_default/orchestrator_config.yml
```

Set the `additional_cloud_init_config_file` parameter to the path of your configuration file:

```yaml title="File: /opt/omnia/orchestrator/input/project_default/orchestrator_config.yml"
additional_cloud_init_config_file: "/opt/omnia/orchestrator/input/project_default/additional_cloud_init.yml"
```

To disable additional cloud-init, leave the value empty:

```yaml title="File: /opt/omnia/orchestrator/input/project_default/orchestrator_config.yml"
additional_cloud_init_config_file: ""
```

### Step 2: Create the additional cloud-init configuration file

A sample is provided in the source at
`src/orchestrator/examples/additional_cloud_init.yml`. Copy it into the
project input directory or create a file with the following structure:

```yaml title="File: /opt/omnia/orchestrator/input/project_default/additional_cloud_init.yml"
---
# Common cloud-init applied to ALL nodes
common:
  write_files:
    - path: /etc/motd
      content: "Welcome to the HPC cluster\n"
      permissions: '0644'
  runcmd:
    - echo "Custom node setup complete" >> /var/log/custom_setup.log

# Per-functional-group cloud-init
groups:
  slurm_node_x86_64:
    runcmd:
      - echo "Slurm compute node initialized" >> /var/log/custom.log
  login_node_x86_64:
    write_files:
      - path: /etc/profile.d/cluster.sh
        content: |
          export CLUSTER_NAME=mycluster
        permissions: '0644'
```

File structure details:

| Key | Type | Description |
|-----|------|-------------|
| `common` | Dictionary | Cloud-init directives applied to all provisioned nodes. |
| `groups` | Dictionary of dictionaries | Each key is a functional group name and must match a `FUNCTIONAL_GROUP_NAME` in the PXE mapping file. Each value contains the cloud-init directives for that group. |

`write_files` entry fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | String | Yes | Absolute path where the file is created on the node. |
| `content` | String | No | Content to write to the file. |
| `permissions` | String | No | File permissions in octal format, for example, `'0644'`. |
| `owner` | String | No | File owner in `user:group` format. |
| `append` | Boolean | No | If `true`, append content to an existing file instead of overwriting. |
| `encoding` | String | No | Content encoding, for example, `base64`. Default is plain text. |

runcmd entries: 
Each entry in the `runcmd` list must be a string. Commands execute during the final stage of cloud-init, after all `write_files` directives are processed.

### Step 3: Run provisioning

```bash title="Run on: OIM"
cd src/main
./omnia.sh --run orchestrator --tags provision
```

The provisioning playbook:

1. Confirms that the configured additional cloud-init file exists. Its
   directives are parsed and merged later in the provisioning workflow.
2. Creates SMD groups for common and per-functional-group cloud-init.
3. Renders and registers the cloud-init configurations with the Boot Script Service (BSS).
4. When nodes PXE boot, cloud-init merges: platform defaults → common additional → per-functional-group additional.

### Merge Behavior

Omnia uses the following cloud-init merge strategy:

- **Dictionaries**: `no_replace` — platform-defined values are not overridden by user entries.
- **Lists**: `append` — user entries (`write_files`, `runcmd`) are appended to platform lists.
- **Order**: Platform defaults are applied first, then common additional cloud-init, then per-functional-group additional cloud-init.

This ensures that platform-critical configurations (networking, boot parameters) are not accidentally overridden.

## Verification

Use the following checks when preparing the file. Only file existence is
currently enforced by the Orchestrator input validator; the remaining rows
describe the supported format and must be reviewed before provisioning:

| Check | Description |
|-------|-------------|
| File existence | The specified configuration file must exist. |
| YAML syntax | Supply valid YAML. Parsing errors surface during provisioning rather than the initial file-existence check. |
| Top-level keys | Use only `common` and `groups` at the top level. |
| Prohibited keys | Do not use `bootcmd`, `network`, `network-config`, or `packages` in any section. |
| Allowed keys | Use only `write_files` and `runcmd` within each section. |
| `write_files` path | Include a `path` in every `write_files` entry. |
| `runcmd` type | Supply every `runcmd` entry as a string. |
| Functional group names | Match keys under `groups` to a `FUNCTIONAL_GROUP_NAME` in the PXE mapping file. |

### Examples

Example 1: Common configuration only

Apply an MOTD banner and a setup script to all nodes:

```yaml title="Example"
common:
  write_files:
    - path: /etc/motd
      content: |
        ========================================
        Dell HPC Cluster - Authorized Users Only
        ========================================
      permissions: '0644'
  runcmd:
    - echo "Node provisioned at $(date)" >> /var/log/provision.log
groups: {}
```

Example 2: Per-functional-group configuration

Run different setup commands on Slurm compute nodes versus login nodes:

```yaml title="Example"
common: {}
groups:
  slurm_node_x86_64:
    runcmd:
      - systemctl enable slurmd
      - echo "Slurm compute node ready" >> /var/log/custom.log
  login_node_x86_64:
    write_files:
      - path: /etc/profile.d/cluster_env.sh
        content: |
          export CLUSTER_NAME=myhpc
          export SCHEDULER=slurm
        permissions: '0644'
    runcmd:
      - echo "Login node ready" >> /var/log/custom.log
```

Example 3: Mixed common and per-group

Apply a common log entry to all nodes, plus additional Slurm-specific commands:

```yaml title="Example"
common:
  runcmd:
    - echo "Cluster node initialized" >> /var/log/custom_setup.log
groups:
  slurm_node_x86_64:
    runcmd:
      - echo "Slurm-specific setup complete" >> /var/log/custom_setup.log
```

In this case, Slurm compute nodes have both the common `runcmd` and the group-specific `runcmd` appended.

## Next steps

- [Provision Nodes](provision_nodes.md) -- Provision the cluster nodes with the updated cloud-init configuration.

## Troubleshooting

- **Custom cloud-init directives not applied on nodes**: Verify that the
  `additional_cloud_init_config_file` parameter in `orchestrator_config.yml`
  contains an existing absolute path, and rerun the Orchestrator `provision`
  tag.

### Limitations

- Customization granularity is at the functional-group level. Per-node cloud-init customization is not supported.
- Only `write_files` and `runcmd` (config and final stage directives) are supported. Early-boot keys remain platform-managed.
- The current input validator checks that the configured file exists, but does
  not enforce all supported keys, value types, or functional-group names.
  Review these requirements before provisioning; unsupported content can fail
  later or produce unintended merged cloud-init.

!!! info

    - [Provision Nodes](provision_nodes.md) -- Run Orchestrator provisioning to apply cloud-init configurations.
    - [Orchestrator contract](../../Reference/domain_contracts/orchestrator_contract.md) -- Orchestrator output paths and runtime behavior.
    - [Additional Cloud Init Reference](../../Reference/Configuration/additional_cloud_init.md) -- Configuration file reference.















