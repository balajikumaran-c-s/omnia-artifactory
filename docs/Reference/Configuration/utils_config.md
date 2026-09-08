# Utils configuration inputs

The current Utils source does not contain a `utils_config.yml` file. Configure
the utility being run through its corresponding input file:

- [install_os_config.yml](install_os_config.md) configures unattended OS installation.
- [collect_pxe.yml](collect_pxe.md) selects nodes for log collection.

Both files are staged under:

```text
$OMNIA_DATA_PATH/utils/input/$OMNIA_PROJECT_NAME/
```

