# Create a mapping file

PXE mapping creation now follows the Discovery-to-Orchestrator domain
contract. Use [Create a mapping file](../discovery/create_mapping_file.md) for
the current manual procedure, schema, valid samples, validation rules, and OME
workflow.

The Orchestrator-owned default is
`$OMNIA_DATA_PATH/orchestrator/input/$OMNIA_PROJECT_NAME/pxe_mapping_file.csv`.
Discovery writes its generated mapping under
`$OMNIA_DATA_PATH/discovery/output/$OMNIA_PROJECT_NAME/`; review and copy it
explicitly before provisioning.
