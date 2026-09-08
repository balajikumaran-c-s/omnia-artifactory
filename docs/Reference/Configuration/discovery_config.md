# discovery_config.yml

The Discovery configuration enables OME discovery and identifies the OME
appliance.

## Location

```text
$OMNIA_DATA_PATH/discovery/input/$OMNIA_PROJECT_NAME/discovery_config.yml
```

The default location is
`/opt/omnia/discovery/input/project_default/discovery_config.yml`.

## Configuration parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `enable_bmc_discovery` | boolean | Yes | Set to `true` to execute node discovery through OME. |
| `ome_ip` | IPv4 string | Yes | OME IPv4 address. When discovery is enabled, it must be valid and must not be a loopback address. |

OME credentials are not stored in this file. The credential workflow creates
the encrypted `discovery_credentials.yml` and its Vault key in the same
project directory.

The Magellan section in the source template is reserved for future
configuration. The current executable Discovery flow uses OME.

## Usage example

```yaml title="File: /opt/omnia/discovery/input/project_default/discovery_config.yml"
enable_bmc_discovery: true
ome_ip: "192.168.1.100"
```

## Related configuration

- [Network specification](network_spec.md)
- [Discovery contract](../domain_contracts/discovery_contract.md)
- [Discover nodes using OME](../../HowTo/discovery/discover_nodes.md)

