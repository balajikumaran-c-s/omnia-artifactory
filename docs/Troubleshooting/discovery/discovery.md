# Discovery issues

Use the Ansible output and `/var/log/omnia/discovery/discovery.log` for every
Discovery failure. The OME execution role also writes
`$OMNIA_DATA_PATH/discovery/output/$OMNIA_PROJECT_NAME/discovery_status.yml`.
Failures during setup, validation, or credential handling occur before that
status file is updated, so an existing file can describe an earlier run.

## Configuration validation fails

???+ note "Symptom"

    Discovery reports an invalid or missing `discovery_config.yml`.

??? note "Resolution"

    - Confirm that the file is in the Discovery project input directory.
    - Retain both `enable_bmc_discovery` and `ome_ip`.
    - When OME discovery is enabled, set `ome_ip` to a valid, non-loopback IPv4
      address.
    - Correct YAML errors and rerun
      `./omnia.sh --run discovery --tags validate` from `src/main`.

## OME is unreachable or authentication fails

???+ note "Symptom"

    Discovery cannot reach OME on port 443 or the OME API rejects the session.

??? note "Resolution"

    - Verify `ome_ip` and connectivity from the OIM to `<ome_ip>:443`.
    - Confirm that OME is running and accessible.
    - Correct the OME username or password in the Discovery credential
      workflow. Rerun `./omnia.sh --run discovery --tags credentials` if a
      stored value must be updated.
    - Keep `.discovery_credentials_key` with an encrypted
      `discovery_credentials.yml`; the files are a matching pair.

## No servers are discovered

???+ note "Cause"

    OME did not return a server device of type `1000` with a nonempty service
    tag.

??? note "Resolution"

    Confirm that the target devices are managed and visible in OME, are
    classified as servers, and report their service tags. Discovery reads the
    existing OME inventory; it does not add devices to OME.

## A server is missing from the mapping

???+ note "Cause"

    The server is assigned to an unsupported nonempty OME static group. Such a
    server remains in the discovery report but is skipped in the mapping.

??? note "Resolution"

    Assign the server to one of the exact supported groups listed in [Plan OME
    static groups](../../HowTo/discovery/discover_nodes.md#plan-ome-static-groups),
    or remove the assignment to use the current default group. Ensure that the
    server belongs to no more than one processed OME group.

## Group or parent values are incorrect

???+ note "Resolution"

    - Make the OME-reported iDRAC hostname contain an `SU...R...` sequence,
      such as `SU1R2OU1C5`. Otherwise Discovery uses `grp0`.
    - For a Slurm compute node, provide a `service_kube_node_x86_64` whose
      iDRAC hostname resolves to the same `GROUP_NAME`. Discovery uses that
      server's service tag as `PARENT_SERVICE_TAG`.
    - Review and correct the generated mapping before handing it to
      Orchestrator.

## NIC or derived IP values are incorrect

???+ note "Resolution"

    Review the timestamped discovery report and the OME network-interface
    inventory. Discovery prefers the first usable non-iDRAC,
    non-InfiniBand Ethernet port reported as `Up`, and falls back to the first
    usable port. It derives the admin and InfiniBand addresses from the first
    two subnet octets and the final two BMC-address octets. The Discovery
    validator does not validate `network_spec.yml`.

## Discovery is blocked by the upgrade lock

???+ note "Resolution"

    Complete the Omnia upgrade before rerunning Discovery. Normal execution is
    blocked while `/opt/omnia/.data/upgrade_in_progress.lock` exists.

!!! info "Related documentation"

    - [Discover nodes using OME](../../HowTo/discovery/discover_nodes.md)
    - [Create a mapping file](../../HowTo/discovery/create_mapping_file.md)
    - [Discovery contract](../../Reference/domain_contracts/discovery_contract.md)
