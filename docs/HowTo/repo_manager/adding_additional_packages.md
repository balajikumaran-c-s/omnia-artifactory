# Add Packages to the Catalog

## Overview

Packages that must be available to an Omnia image are selected in the catalog
identified by `CATALOG_FILE_PATH`.

Use the `catalog_add` operation to add or update RPM packages, RPM repository
packages, tarballs, or container images. The operation uses upsert semantics:
it creates missing groups, updates existing package definitions, and avoids
duplicate component references.

## Prerequisites

- Create and synchronize the local repositories as described in
  [Create Local Repositories](configure_repos.md).
- Load the Omnia environment and activate the shared virtual environment.
- Identify the existing catalog functional layer and group that should own the
  package.
- For an RPM, identify the exact `reponame` and ensure it is mapped under the
  matching catalog version and architecture in `repo_manager_config.yml`.
- For an image from a non-public registry, configure the registry and its
  credentials before adding the image.

## Procedure

1. Create an INI-like additions file. This example adds two RPMs to an existing
   group and ensures the group is part of an existing functional layer:

    ~~~ini
    [defaults]
    arch=x86_64, os=rhel, os_version=10.0

    [openldap_group | description=OpenLDAP packages]
    openldap, rpm, openldap, baseos
    openldap_clients, rpm, openldap-clients, baseos

    [slurm_control_node_rhel_10_0_x86_64 | type=functional_layer]
    "openldap_group"
    ~~~

    Replace `slurm_control_node_rhel_10_0_x86_64` with the exact
    functional-layer name
    from your catalog. A new functional layer must include exactly one
    `base_os` group; adding a package group to an existing valid layer keeps
    its existing base OS reference.

    Supported input-line formats for `catalog_add` are:

    | Content | Format |
    |---|---|
    | RPM | `key, rpm, package_name, reponame` |
    | Tarball | `key, tarball, artifact_name, https_url` |
    | Container image | `key, image, registry/image_path, registry, tag` |

    A line can end with `arch=`, `os=`, or `os_version=` overrides.

2. Add the entries to the configured catalog:

    ~~~bash title="Run on: OIM host"
    cd <OMNIA_SOURCE_PATH>/src/repo_manager/playbooks
    ansible-playbook repo_manager.yml --tags catalog_add \
      -e "input_file=/absolute/path/to/additions.txt"
    ~~~

    By default, the command reads and writes the catalog selected by
    `CATALOG_FILE_PATH` and validates the result. Use `catalog_input` and
    `output_file` extra variables when the change must be reviewed in a
    separate output file.

3. Validate the selected catalog and its source mappings:

    ~~~bash title="Run on: OIM host"
    ansible-playbook repo_manager.yml --tags catalog_validate
    ansible-playbook repo_manager.yml --tags precheck
    ~~~

4. Synchronize the changed catalog and regenerate the consumer contract:

    ~~~bash title="Run on: OIM host"
    ansible-playbook repo_manager.yml --tags "download,status"
    ~~~

## Verification

Check that the new package is present in the catalog, then review its group
status:

~~~bash title="Run on: OIM host"
python3 -m json.tool "$CATALOG_FILE_PATH"
sed -n '1,240p' \
  /opt/omnia/repo_manager/output/project_default/repo_status.yml
~~~

Per-package results are written to
`<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/<architecture>/<group>/status.csv`.
Confirm that the added entries report `Success` and that
`repo_status.yml` has `overall_status: success`.

## Next steps

- [Build Cluster Images](../image_build_manager/build_images.md).
- [Add a repository mapping](adding_additional_repositories.md) for a new RPM
  source.
- [Configure content types](configuring_specific_software.md) for image,
  Python, or file artifacts.

## Troubleshooting

- **The additions file fails to parse**: Use one supported positional format,
  put package lines below a group header, and put group references below a
  `type=functional_layer` header.
- **The updated catalog fails validation**: Check that every functional layer
  has exactly one `base_os` group and that all group and package references
  exist.
- **An RPM mapping is missing**: Match the source version, architecture, and
  `reponame` in `repo_manager_config.yml`.
- **An `rpm_repo` entry is rejected**: Its repository must retain content;
  change the effective policy so it does not resolve to `streamed`.
- **The package is not processed**: Ensure its group is referenced by a
  selected functional layer. Orphan groups and packages are not selected.
