# Add Nodes through BuildStreaM

## Overview

Add nodes by updating the Orchestrator PXE mapping in the managed GitLab
project and running the BuildStreaM deploy pipeline. A committed change to
`input/orchestrator/pxe_mapping_file.csv` automatically selects the deploy
pipeline. The child pipeline then requires an operator to select a `BUILT`
image group and start deployment.

The mapping is uploaded to Orchestrator as the desired deployment input. Keep
all nodes that must remain in the cluster in the file; the BuildStreaM source
does not create a new-node-only inventory from the changed rows.

## Prerequisites

- Complete [Deploy GitLab and BuildStreaM](../../HowTo/build_stream/deploy_gitlab.md).
- Ensure at least one image group has `BUILT` status. Run the
  [build pipeline](../../HowTo/build_stream/execute_build_pipeline.md) first if
  no built image group is available.
- Ensure the new physical nodes are powered on and their BMC and admin
  addresses are reachable from the OIM.
- Collect every value required by the source mapping header and use functional
  group names present in the selected catalog.

## Procedure

1. In the managed GitLab project, open
   `input/orchestrator/pxe_mapping_file.csv`, retain all existing rows, append
   the new node rows, and commit the change.

    ```csv title="input/orchestrator/pxe_mapping_file.csv"
    FUNCTIONAL_GROUP_NAME,GROUP_NAME,SERVICE_TAG,PARENT_SERVICE_TAG,HOSTNAME,ADMIN_MAC,ADMIN_IP,BMC_MAC,BMC_IP,IB_NIC_NAME,IB_IP
    <catalog-functional-group>,grp1,79WWJ95,,new-node1,<admin-mac>,172.16.107.50,<bmc-mac>,172.17.107.50,,
    <catalog-functional-group>,grp1,79WWJ96,,new-node2,<admin-mac>,172.16.107.51,<bmc-mac>,172.17.107.51,,
    ```

    Replace the placeholders with values for the target hardware and a
    functional-group name present in the selected catalog.

2. Open **Build** -> **Pipelines**. Confirm that the commit started the deploy
   parent pipeline.

3. Open its downstream pipeline and run the manual selection job for the
   required `BUILT` image group.

4. Run the manual `deploy` job. The child pipeline uploads the current
   Orchestrator inputs and then runs the `deploy`, `restart`, `validate`, and
   `summary` stages in sequence.

For the complete pipeline procedure, see
[Execute the deploy pipeline](../../HowTo/build_stream/execute_deploy_pipeline.md).

## Verification

1. Confirm that the downstream `deploy`, `restart`, `validate`, and `summary`
   jobs complete successfully.
2. In the summary output, confirm the selected Job ID and Image Group ID and
   verify that the three operational stages report `COMPLETED`.
3. Confirm that each added node is reachable on its admin IP and is registered
   in the intended Slurm or Kubernetes cluster.

## Next steps

- Keep the committed mapping synchronized with the intended cluster inventory.
- Use the direct [Orchestrator add-node procedure](../add_nodes.md) when a
  custom new-node-only PXE inventory is required.
- [Clean up old image groups](cleanup_operations.md) when they are no longer
  required.

## Troubleshooting

- **No deploy pipeline starts:** Confirm the committed path is exactly
  `input/orchestrator/pxe_mapping_file.csv`.
- **No image-selection jobs appear:** The deploy pipeline lists only image
  groups with `BUILT` status. Complete a build pipeline first.
- **An operational stage fails:** Open its GitLab job log and use the BSM job
  log path reported for the failed stage.
- See [BuildStreaM troubleshooting](../../Troubleshooting/build_stream/build_stream.md)
  for additional investigations.
