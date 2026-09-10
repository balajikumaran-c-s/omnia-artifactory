# Image Build Manager Domain Contract

**Deployment module**: Image Build Manager | **CLI identifier**: `image_build_manager`

## Upstream domain contract

Image Build Manager consumes `repo_status.yml`, the output contract produced
by Repository Manager. Build-related flows require the file and validate it
against the Repo Manager status schema before loading repository data.

The required contract includes a successful overall status, operating-system
metadata, versioned repository mappings, and Repo Manager certificate data.
When a certificate path is present, the certificate must also exist. The
prepare, validation, precheck, and cleanup flows do not require this upstream
output.

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
use this recorded value, rather than current runtime settings, to interpret
the artifact layout.

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
