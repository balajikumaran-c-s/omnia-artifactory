# Omnia Documentation

[![Omnia version](https://img.shields.io/github/v/release/dell/omnia?include_prereleases)](https://github.com/dell/omnia/releases)
[![Downloads](https://img.shields.io/github/downloads/dell/omnia/total)](https://github.com/dell/omnia/releases)
[![Last Commit](https://img.shields.io/github/last-commit/dell/omnia)](https://github.com/dell/omnia/commits)
[![Contributors](https://img.shields.io/github/contributors/dell/omnia)](https://github.com/dell/omnia/graphs/contributors)
[![Forks](https://img.shields.io/github/forks/dell/omnia)](https://github.com/dell/omnia/network/members)
[![License](https://img.shields.io/github/license/dell/omnia)](https://github.com/dell/omnia/blob/main/LICENSE)

Omnia is an open-source deployment toolkit for building and managing HPC and
AI infrastructure on Linux-based Dell PowerEdge servers. From an Omnia
Infrastructure Manager (OIM), administrators can synchronize software content,
build stateless node images, discover hardware, provision Slurm and service
Kubernetes clusters, deploy telemetry, and run supporting lifecycle utilities.

## Modular deployment architecture

Omnia uses a modular, capability-based deployment architecture. Each
**deployment module** owns its configuration, Ansible entry playbook, runtime
data, logs, and outputs. Modules exchange documented input/output contracts
such as YAML status files, JSON catalogs, and CSV node mappings.

The `src/main/omnia.sh` script prepares the common runtime and invokes one
module at a time. The module's CLI value and source-directory name remain its
internal domain identifier. `main` coordinates setup and execution; it is not
a deployment module.

| Deployment module | Customer outcome | Typical dependency |
|---|---|---|
| `repo_manager` | Deploy Pulp and synchronize catalog-selected RPMs, images, files, and Python content. | None |
| `image_build_manager` | Deploy image storage services and build functional-group OS images. | Successful Repository Manager output |
| `discovery` | Discover BMC endpoints through OME and produce a PXE mapping. | Independent and optional when a mapping is supplied manually |
| `orchestrator` | Deploy OpenCHAMI and optional OpenLDAP, provision Slurm or service Kubernetes, and optionally start physical nodes through iDRAC PXE boot. | Built images, repository information, and a PXE mapping |
| `telemetry` | Deploy enabled telemetry sources, bridges, and sinks on service Kubernetes. | A provisioned service Kubernetes cluster |
| `build_stream` | Deploy BuildStreaM Manager, GitLab integration, and catalog-driven build and deploy pipelines. | Prepared Repository Manager and Image Build Manager services |
| `utils` | Run utilities such as cluster-log collection, OIM log backup, and unattended OS installation. | Depends on the selected utility |

For a direct deployment, the source-defined order is:

```text
Repository Manager -> Image Build Manager -> optional Discovery
                   -> Orchestrator -> optional Telemetry
```

BuildStreaM provides a separate automation path: its build pipeline invokes
Repository Manager and Image Build Manager, and its deploy pipeline invokes
Orchestrator. Utils is used independently when an operational task requires it.

See [Architecture](Overview/architecture.md),
[Running Deployment Modules](Overview/domain_execution.md), and
[Module Contracts](Reference/index.md#module-contracts) for the detailed
interfaces and handoffs.

## Choose a deployment path

<div class="grid cards" markdown>

-   :material-server: **[Slurm Quickstart](GetStarted/slurm_quickstart.md)**

    ---

    Build Slurm images and provision a Slurm cluster.

-   :material-kubernetes: **[Kubernetes and Telemetry](GetStarted/k8s_telemetry_only.md)**

    ---

    Provision service Kubernetes and deploy selected non-LDMS telemetry
    integrations without Slurm.

-   :material-view-dashboard: **[Full Deployment](GetStarted/full_deployment.md)**

    ---

    Provision Slurm and service Kubernetes, then deploy the required Telemetry
    sources and sinks.

-   :material-source-branch: **[BuildStreaM](GetStarted/buildstream_deployment.md)**

    ---

    Use GitLab pipelines to synchronize catalog content, build images, and
    deploy mapped nodes.

</div>

## Documentation map

<div class="grid cards" markdown>

-   :material-book-open-variant: **[Overview](Overview/index.md)**

    ---

    Learn the architecture, component responsibilities, network topologies,
    module execution model, and terminology.

-   :material-rocket-launch: **[Get Started](GetStarted/index.md)**

    ---

    Select a supported deployment path and follow its required module
    sequence.

-   :material-tools: **[How-to Guides](HowTo/index.md)**

    ---

    Complete a task within `main`, Repository Manager, Image Build Manager,
    Discovery, Orchestrator, Telemetry, BuildStreaM, or Utils.

-   :material-file-document: **[Reference](Reference/index.md)**

    ---

    Look up configuration files, module contracts, support matrices, samples,
    and module playbook entry points.

-   :material-cog: **[Operations & Maintenance](Operations/index.md)**

    ---

    Perform day-2 repository, node, BuildStreaM, diagnostic, and platform
    lifecycle operations.

-   :material-alert-circle: **[Troubleshooting](Troubleshooting/index.md)**

    ---

    Diagnose cross-module and module-specific failures.

</div>

## Before deployment

Complete the [Prerequisites Checklist](GetStarted/prerequisites_checklist.md),
then review the prerequisites for every deployment module included in the
selected path. A deployment does not require every module or every optional telemetry
source.

## Licensing

Omnia is available under the [Apache 2.0 license](https://opensource.org/licenses/Apache-2.0).
Omnia deploys open-source and third-party software that remains subject to its
own licenses. See the
[installed software matrix](Reference/SupportMatrix/installed_software.md) for
the applicable components and licenses.

## Omnia community members

<div class="community-logos" style="display: flex; flex-wrap: wrap; align-items: center; gap: 2rem; margin: 1rem 0;">
  <a href="https://www.dell.com"><img src="assets/images/delltech.png" alt="Dell Technologies" style="height: 60px;"></a>
  <a href="https://www.intel.com"><img src="https://upload.wikimedia.org/wikipedia/commons/0/0e/Intel_logo_%282020%2C_light_blue%29.svg" alt="Intel" style="height: 40px;"></a>
  <a href="https://www.unipi.it"><img src="assets/images/pisa.png" alt="University of Pisa" style="height: 60px;"></a>
  <img src="https://user-images.githubusercontent.com/83095575/117071024-64956c80-ace3-11eb-9d90-2dac7daef11c.png" alt="Community Member" style="height: 60px;">
  <img src="https://images.squarespace-cdn.com/content/v1/660f1a48587dbb2769709a33/9ac5520f-a308-4751-80f4-415d07a23473/VIZIAS+Blue.png" alt="VIZIAS" style="height: 60px;">
  <img src="https://user-images.githubusercontent.com/5414112/153955170-0a4b199a-54f0-42af-939c-03eac76881c0.png" alt="Community Member" style="height: 60px;">
  <a href="https://www.liqid.com"><img src="assets/images/Liqid.png" alt="Liqid" style="height: 50px;"></a>
</div>

If you have feedback about the Omnia documentation, contact
[omnia.readme@dell.com](mailto:omnia.readme@dell.com).
