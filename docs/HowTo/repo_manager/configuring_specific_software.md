# Configure Catalog Content

## Overview

Repo Manager selects software from the catalog's
`functional layer -> group -> package -> source` hierarchy. It does not contain
a customer-facing list of software names or versions. To make software
available for an Omnia image, define the exact upstream content in the catalog
and connect it to a selected functional layer.

The runtime supports these content families:

| Catalog `packagetype` | Source selection | Pulp content |
|---|---|---|
| `rpm`, `rpm_repo`, `rpm_file` | Repository mapping or direct RPM source | RPM |
| `image` | Image name, tag, and registry | Container |
| `pip_module` | Python package and version | Python |
| `tarball`, `manifest`, `git`, `iso`, `shell`, `ansible_galaxy_collection` | Type-specific HTTP(S) source | File |

The catalog generate/add text interface supports `rpm`, `rpm_repo`,
`tarball`, and `image`. Other runtime-supported types must already be
present in an approved catalog.

## Prerequisites

- Complete [Create Local Repositories](configure_repos.md).
- Know which functional layer and architecture will consume the software.
- Identify a reachable upstream repository, registry, or artifact URL.
- For a private registry, configure its authentication and TLS mapping in
  `repo_manager_config.yml`.

## Procedure

1. Choose the content type and define its package source in the catalog.

    RPM example:

    ~~~json
    {
      "name": "bash",
      "packagetype": "rpm",
      "sources": [
        {
          "architecture": "x86_64",
          "name": "rhel",
          "version": ["10.0"],
          "reponame": "baseos"
        }
      ]
    }
    ~~~

    Container image example:

    ~~~json
    {
      "name": "registry.k8s.io/kube-controller-manager",
      "packagetype": "image",
      "tag": "v1.35.1",
      "sources": [
        {
          "architecture": "x86_64",
          "registry": "registry.k8s.io",
          "name": "rhel",
          "version": ["10.0"]
        }
      ]
    }
    ~~~

    Python packages can use either `name: "cffi==1.17.1"` or `name: "cffi"`
    with `version: "1.17.1"`. If both forms provide a version, the versions
    must match.

2. Add the package key to a catalog group and add that group to each functional
   layer that requires the content. Every functional layer must include exactly
   one `base_os` group.

3. Map each RPM `reponame` and each non-public image `registry` in
   `repo_manager_config.yml`. Repository lookup is independent for every OS
   minor version and architecture.

4. Validate the catalog and mappings:

    ~~~bash title="Run on: OIM host"
    cd <OMNIA_SOURCE_PATH>/src/repo_manager/playbooks
    ansible-playbook repo_manager.yml --tags catalog_validate
    ansible-playbook repo_manager.yml --tags precheck
    ~~~

5. Synchronize the selected content and regenerate status:

    ~~~bash title="Run on: OIM host"
    ansible-playbook repo_manager.yml --tags "download,status"
    ~~~

## Verification

Review the package row in:

~~~text
<REPO_MANAGER_DATA_PATH>/log/<os>/<version>/<architecture>/<group>/status.csv
~~~

Then confirm `overall_status: success` in
`<REPO_MANAGER_DATA_PATH>/output/<project>/repo_status.yml`. RPM URLs appear
under `repositories`; File and Python content appears under `file_repos`;
configured registry metadata appears under `registries`.

## Next steps

- [Build Cluster Images](../image_build_manager/build_images.md).
- [Add Packages to the Catalog](adding_additional_packages.md) with the
  supported text input format.
- [Update Local Repositories](../../Operations/repo_manager/updating_local_repositories.md) after changing an
  existing catalog definition.

## Troubleshooting

- **Content is not selected**: Verify that the package is referenced by a group
  and that the group is referenced by a functional layer.
- **A direct artifact URL is rejected**: Use HTTP(S) without embedded
  credentials, fragments, malformed escapes, or credential-bearing query keys.
- **A private image is rejected**: The catalog source must use the configured
  registry key, but the image name must begin with the registry's actual
  `host[:port]`.
- **Multiple versions do not all run**: Each selected OS context needs a
  matching source version and architecture, or an explicit `noarch` source.
  Repo Manager stops later contexts if an earlier one fails.
