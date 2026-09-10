# Make Additional Content Available to Image Builds

## Overview

Repo Manager synchronizes catalog-selected content into Pulp and publishes its
locations in `repo_status.yml`. It does not install RPMs or configure container
runtimes on cluster nodes. Image Build Manager consumes the successful Repo
Manager contract when it builds an image.

Use this workflow when an image needs content beyond what the current catalog
functional layer selects.

## Prerequisites

- Complete [Create Local Repositories](configure_repos.md).
- Identify the image's catalog functional layer and target architecture.
- Identify the upstream RPM repository, container registry, Python package, or
  file source.
- Ensure the downstream Image Build Manager uses the same catalog context and
  Repo Manager output.

## Procedure

1. Add the required package to the catalog and to a group referenced by the
   image's functional layer. See
   [Configure Catalog Content and Add Packages](adding_additional_packages.md).

2. Add any required source mapping:

   - Map RPM `reponame` values under the matching version and architecture.
   - Configure non-public registries under `registries`.
   - Keep registry credentials in the generated Ansible Vault credential file,
     not in the catalog or main configuration.

3. Validate the catalog and input mappings:

    ~~~bash title="Run on: OIM host"
    cd <OMNIA_SOURCE_PATH>/src/repo_manager/playbooks
    ansible-playbook repo_manager.yml --tags catalog_validate
    ansible-playbook repo_manager.yml --tags precheck
    ~~~

4. Synchronize the content and generate a fresh consumer contract:

    ~~~bash title="Run on: OIM host"
    ansible-playbook repo_manager.yml --tags "download,status"
    ~~~

5. Start the Image Build Manager workflow only after the generated
   `repo_status.yml` reports success.

## Verification

Check the added content's package status:

~~~text
<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/<architecture>/<group>/status.csv
~~~

Then inspect
`<REPO_MANAGER_DATA_PATH>/output/<project>/repo_status.yml`:

- `overall_status` is `success`.
- `overall_status_by_version` is `success` for every selected context.
- Required RPM URLs appear below `repositories`.
- File and Python distribution URLs appear below `file_repos`.
- Pulp certificate paths appear below `repo_manager.certificates`.

## Next steps

- [Build Cluster Images](../image_build_manager/build_images.md).
- [Configure Catalog Content and Add Packages](adding_additional_packages.md).
- [Add an RPM Repository](adding_additional_repositories.md).

## Troubleshooting

- **Image Build Manager cannot find the content**: Confirm Repo Manager wrote a
  successful status file for the same project, OS version, and architecture.
- **An RPM is missing**: Verify the catalog package is in a selected group and
  its `reponame` maps to a synchronized distribution.
- **An image is missing**: Check its exact image name and tag and the catalog
  source's registry mapping. Multiple tags are tracked independently.
- **The status file is stale**: Run the `status` tag after synchronization.
  Selective cleanup removes stale status output intentionally.
