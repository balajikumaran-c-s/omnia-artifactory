# Image Build Manager

## Overview

Image Build Manager builds RHEL or Rocky Linux images for `x86_64` and
`aarch64` HPC cluster provisioning by using OpenCHAMI. It deploys MinIO S3 and
a local OCI registry, builds an image for each functional group, and writes
`build_status.yml` for the provisioning workflow.

Image Build Manager runs on the Omnia Infrastructure Manager (OIM). Tasks run
locally except for `aarch64` builds, which use SSH to run on a remote ARM host.

```text
  repo_status.yml                                                    build_status.yml
  catalog_rhel.json (or package_groups.yml)                          S3 artifacts
  +---------------------+     +-------------------------------------+     +-----------+
  |                     |     |       Image Build Manager            |     |           |
  |  repo_manager       |---->|                                     |---->| provision |
  |  (upstream)         |     |  setup -> validate -> prepare       |     | workflow  |
  |                     |     |         -> build -> write_status    |     | (consumer)|
  +---------------------+     +-------------------------------------+     +-----------+
                                       |              |
                                  MinIO S3      OCI Registry
                                 (boot-images)  (+ regctl)
```

## Prerequisites

| Requirement | Minimum | Validated |
|---|---|---|
| OIM operating system | RHEL 10.x or Rocky Linux 10.x | RHEL 10.0 |
| Python | 3.12+ | 3.12.8 |
| Ansible | `ansible-core` 2.20+ | 2.20.0 |
| Podman | 5.0+ | 5.3.1 |
| Free disk space | 50 GB | Not specified |

Build operations also require a successful `repo_status.yml` from Repo Manager.
An `aarch64` build requires a separate, reachable ARM host because
cross-architecture builds are not supported.

## Choose a task

| Task | Use it to |
|---|---|
| [Build OS Images](build_images.md) | Configure the build inputs, prepare storage and registry services, build functional-group images, upload their artifacts, and generate `build_status.yml`. |

## Contract reference

See the [Image Build Manager Input/Output Contract](../../Reference/domain_contracts/image_build_manager_contract.md)
for the configuration, credentials, upstream Repo Manager contract, package
sources, generated `build_status.yml`, services, and S3 artifact layout.

After Image Build Manager produces a successful `build_status.yml`, the
[provisioning workflow](../orchestrator/provision_nodes.md) can consume its
functional-group image paths.
