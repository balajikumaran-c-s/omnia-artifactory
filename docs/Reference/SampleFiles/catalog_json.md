# Catalog JSON reference

The catalog JSON defines the functional layers, software packages, and artifact sources used by Omnia when preparing repositories, building operating-system images, and provisioning the cluster.

The default installed catalog path is:

```text
${OMNIA_DATA_PATH}/catalog/catalog_rhel.json
```

When `OMNIA_DATA_PATH` uses its default value, the path is `/opt/omnia/catalog/catalog_rhel.json`.

The default catalog in the Omnia source is `src/main/samples/catalog_rhel.json`.

For complete catalog files, see the [Omnia catalog samples](https://github.com/dell/omnia/tree/issue-4849-omnia-modernization/src/main/samples/catalogs).
