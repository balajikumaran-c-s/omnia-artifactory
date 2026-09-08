# package_groups.yml

This file supplies OS metadata and RPM package mappings for Image Build Manager
when `functional_groups_source: config` is set in `image_build_config.yml`.

## Location

```text
$OMNIA_DATA_PATH/image_build_manager/input/$OMNIA_PROJECT_NAME/package_groups.yml
```

## Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `os` | string | Yes | Target operating-system name, such as `rhel`. |
| `os_version` | string | Yes | Numeric dotted version, such as `10.0`. |
| `base_packages` | list of strings | Yes | Unique, non-empty RPM names installed in every image. |
| `functional_groups` | object | Yes | One or more functional-group definitions. |
| `functional_groups.<name>.packages` | list of strings | Yes | Unique RPM names added to that functional-group image. The list may be empty. |

Functional-group names must end in `_x86_64` or `_aarch64` and can otherwise
contain letters, digits, underscores, and hyphens. Unknown fields are rejected.

## Usage example

```yaml title="File: /opt/omnia/image_build_manager/input/project_default/package_groups.yml"
os: "rhel"
os_version: "10.0"

base_packages:
  - systemd
  - kernel
  - dracut
  - nfs-utils

functional_groups:
  os_x86_64:
    packages: []
  slurm_node_x86_64:
    packages:
      - munge
      - slurm-slurmd
```

This file is not used for package resolution when
`functional_groups_source: catalog` is selected.

## Related configuration

- [Image Build configuration](image_build_manager_config.md)
