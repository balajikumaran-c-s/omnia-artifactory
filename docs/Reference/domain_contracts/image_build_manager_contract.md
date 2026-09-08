# Image Build Manager Input/Output Contract

**Deployment module**: Image Build Manager | **CLI identifier**: `image_build_manager` | **Collection**: `omnia.image_build`

## Input contract

Image Build Manager stages its project inputs under
`<IMAGE_BUILD_MANAGER_DATA_PATH>/input/<project>/`. When
`IMAGE_BUILD_MANAGER_DATA_PATH` is unset, the runtime root defaults to
`<OMNIA_DATA_PATH>/image_build_manager`.

### `image_build_config.yml`

**Purpose**: Configures the repository dependency, S3 provider, build engine,
package source, build controls, and optional ARM host.

**Schema**:
`plugins/module_utils/input_validation/schema/image_build_config.json`

| Field | Type | Required | Default | Purpose |
|---|---|---|---|---|
| `repo_manager_output_path` | string | Yes | Project-dependent | Full path to Repo Manager's `repo_status.yml`. |
| `s3_configurations.provider` | string | Yes | `minio` | Selects local `minio` or external `powerscale`. |
| `s3_configurations.endpoint_url` | string | Yes | Empty | Must be empty for MinIO and a valid HTTP(S) URL for PowerScale. |
| `image_build_type` | string | Yes | `image-thrillhouse` | Selects `image-builder` or `image-thrillhouse`. |
| `functional_groups_source` | string | Yes | `config` | Selects `package_groups.yml` or catalog JSON as the package source. |
| `build_image.max_parallel` | integer | Yes | `0` | Maximum parallel image builds; `0` allows all groups concurrently. |
| `build_image.build_timeout` | integer | Yes | `7200` | Per-build timeout from 600 through 86400 seconds. |
| `build_image.force_rebuild` | boolean | Yes | `false` | Bypasses the package-hash cache. |
| `build_image.backup_s3_images` | boolean | Yes | `false` | Copies existing compute artifacts to `*_prev` before rebuilding. |
| `build_image.repo_ssl_verify` | boolean | Yes | `true` | Enables repository SSL verification and GPG checks. |
| `aarch64_inventory_host_ip` | IPv4 string | No | Empty | Selects a remote ARM build host; empty skips `aarch64`. |
| `aarch64_ssh_user` | string | Conditional | `root` | SSH user required when an ARM host is selected. |

Unknown fields are rejected by the schema.

### `image_build_credentials.yml`

**Purpose**: Stores S3 and optional ARM-host credentials.

**Schema**:
`plugins/module_utils/input_validation/schema/image_build_credentials.json`

The credential file is generated through interactive collection and encrypted
with Ansible Vault. Its key is stored alongside it as
`.image_build_credentials_key`.

| Field | Required | Purpose |
|---|---|---|
| `s3_secret_key` | Yes | MinIO password or PowerScale S3 secret key. |
| `s3_access_id` | For PowerScale | S3 access key ID. |
| `aarch64_ssh_password` | When an ARM host is configured | Password used to establish SSH access. |

### `repo_status.yml`

**Purpose**: Provides RPM repository URLs, OS metadata, and certificate paths
from Repo Manager.

**Location**: The path configured by `repo_manager_output_path`.

The consumer contract requires `overall_status: success`, an operating-system
type, Repo Manager metadata, and versioned repository maps. A configured Repo
Manager certificate must exist. Build-related flows require this file;
validation, preparation, precheck, and cleanup flows do not.

### `package_groups.yml`

This file is used when `functional_groups_source` is `config`.

| Field | Required | Purpose |
|---|---|---|
| `os` | No | Sets the build operating-system type. |
| `os_version` | No | Sets the build operating-system version. |
| `base_packages` | Yes | RPM packages installed in every image. |
| `functional_groups.<name>.packages` | Yes | Additional RPM packages for a functional-group image. |

Functional-group keys are filtered by the `_x86_64` or `_aarch64` suffix. The
group list comes from the keys in this file; it is not duplicated in
`image_build_config.yml`.

### Catalog JSON

Catalog JSON is used when `functional_groups_source` is `catalog` and is read
from `CATALOG_FILE_PATH`. Image Build Manager consumes
`catalog.identifier`, `catalog.functionallayer`, `catalog.groups`, and
`catalog.packages`.

Layer names beginning with `baseos` provide the base-image packages. Other
layers matching the selected architecture provide functional-group images.
The operating-system type and version come from the base OS group.

## Output contract

### `build_status.yml`

**Location**:
`<IMAGE_BUILD_MANAGER_DATA_PATH>/output/<project>/build_status.yml`

**Producer**: The `build_os_images` role's status-writing task.

**Consumer**: The provisioning workflow, for image validation and BSS template
rendering.

| Field | Type | Purpose |
|---|---|---|
| `overall_status` | string | Reports `success` or `failed`. |
| `image_build_type` | string | Records the producing engine: `image-builder` or `image-thrillhouse`. |
| `s3_configurations.endpoint_url` | string | S3 HTTP(S) endpoint without an artifact path. |
| `s3_configurations.bucket` | string | Artifact bucket, currently `boot-images`. |
| `functional_group_images[].functional_group` | string | Functional-group name with its architecture suffix. |
| `functional_group_images[].kernel` | string | Endpoint-relative kernel object path. |
| `functional_group_images[].initrd` | string | Endpoint-relative initramfs object path. |
| `functional_group_images[].image` | string | Endpoint-relative root-filesystem object path. |

Each artifact path includes the bucket name, omits the endpoint and `s3://`
scheme, and ends with the object filename. Consumers construct a download URL
as `<s3_configurations.endpoint_url>/<artifact-path>`.

`image_build_type` records the engine that produced the manifest. Consumers
use this value, rather than the current input configuration, to interpret the
artifact layout.

### S3 artifact layouts

`image-builder` publishes:

```text
boot-images/efi-images/<functional_group>/<image_name>-imgbld/vmlinuz-<kernel-version>
boot-images/efi-images/<functional_group>/<image_name>-imgbld/initramfs-<kernel-version>.img
boot-images/<functional_group>/<image_name>-imgbld/<rootfs-filename>
```

`image-thrillhouse` publishes:

```text
boot-images/<functional_group>/<image_name>-imgth/<release>/vmlinuz
boot-images/<functional_group>/<image_name>-imgth/<release>/initramfs.img
boot-images/<functional_group>/<image_name>-imgth/<release>/rootfs.squashfs
```

The `efi-images` segment is an object-key prefix inside the `boot-images`
bucket, not a separate bucket.

### Deployed services

| Service | Condition | Endpoint |
|---|---|---|
| `minio.service` | S3 provider is not PowerScale | Ports 9000 for the API and 9001 for the console. |
| `registry.service` | Always during preparation | Port 5000 over HTTP. |

Both services are deployed as Podman Quadlets and added to `omnia.target`.

### Cleanup

The full Image Build Manager cleanup removes the MinIO and registry containers
and data, `build_status.yml`, S3 buckets and artifacts, service entries,
credentials, and the `s3cmd` configuration.

## Related documentation

- [Image Build Manager](../../HowTo/image_build_manager/index.md)
- [Build OS Images](../../HowTo/image_build_manager/build_images.md)
- [Repository Manager Contract](repo_manager_contract.md)
