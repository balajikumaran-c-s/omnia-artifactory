# Configure Base OS and Functional Package Groups

## Overview

The catalog defines all image content. A group with `type: "base_os"` contains
the foundational RPMs for an OS version, while ordinary groups contain the
packages for a functional area.

Every functional layer must reference exactly one `base_os` group. A package
is processed only when it is reachable from a selected functional layer.

## Prerequisites

- Prepare a catalog that follows the Repo Manager catalog structure.
- Identify the base packages required for the target RHEL minor version and
  architecture.
- Ensure each package's `reponame` exists in the matching
  `repo_manager_config.yml` repository map.

## Procedure

1. Define a base OS group and its packages. The catalog generator accepts this
   input format:

    ~~~ini
    [defaults]
    arch=x86_64, os=rhel, os_version=10.0

    [baseos_group_10.0 | type=base_os, description=Base OS packages, os=rhel, os_version=10.0]
    systemd, rpm, systemd, baseos
    systemd_udev, rpm, systemd-udev, baseos
    glibc_langpack_en, rpm, glibc-langpack-en, baseos
    ~~~

2. Define an ordinary functional group:

    ~~~ini
    [openldap_group | description=OpenLDAP packages]
    openldap, rpm, openldap, baseos
    openldap_clients, rpm, openldap-clients, baseos
    ~~~

3. Reference both groups from the intended functional layer:

    ~~~ini
    [slurm_control_node_rhel_10_0_x86_64 | type=functional_layer]
    "baseos_group_10.0"
    "openldap_group"
    ~~~

4. Generate a new catalog, or add these definitions to an existing one:

    ~~~bash title="Run on: OIM host"
    cd <OMNIA_SOURCE_PATH>/src/repo_manager/playbooks

    # Generate fails if the output exists unless force=true.
    ansible-playbook repo_manager.yml --tags catalog_generate \
      -e "input_file=/absolute/path/to/catalog-input.txt" \
      -e "output_file=/absolute/path/to/catalog.json" \
      -e "catalog_name=my_catalog"

    # For an existing catalog, use catalog_add instead.
    ansible-playbook repo_manager.yml --tags catalog_add \
      -e "input_file=/absolute/path/to/additions.txt"
    ~~~

5. Set `CATALOG_FILE_PATH` to the approved output, then validate and synchronize:

    ~~~bash title="Run on: OIM host"
    export CATALOG_FILE_PATH=/absolute/path/to/catalog.json
    ansible-playbook repo_manager.yml \
      --tags "precheck,download,status"
    ~~~

## Verification

Run catalog validation and confirm there are no referential-integrity or
business-rule errors:

~~~bash title="Run on: OIM host"
ansible-playbook repo_manager.yml --tags catalog_validate
~~~

Inspect the base group and ordinary group results under
`<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/<architecture>/`, and confirm the
generated `repo_status.yml` has `overall_status: success`.

## Next steps

- [Build Cluster Images](../image_build_manager/build_images.md).
- [Configure Catalog Content](configuring_specific_software.md).
- [Add Packages to the Catalog](adding_additional_packages.md).

## Troubleshooting

- **A functional layer has no or multiple base groups**: Reference exactly one
  group whose `type` is `base_os`.
- **A group or package is orphaned**: Add the group to a functional layer and
  the package to a group. Orphan definitions generate warnings and are not
  selected for synchronization.
- **A package cannot be resolved**: Check its source architecture, version, and
  `reponame` against `repo_manager_config.yml`.
- **Catalog generation would overwrite a file**: Write to a new
  `output_file`, or use `force=true` only after reviewing the target.
