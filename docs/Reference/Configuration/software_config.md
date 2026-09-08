# software_config.json is not a current module input

The current Repo Manager and Orchestrator initialization flows do not stage a
customer-authored `software_config.json`. Software selection is catalog-driven,
and Image Build Manager can alternatively read
[package_groups.yml](package_groups.md) when
`functional_groups_source: config` is selected.

Some Build Stream adapter and compatibility paths can generate or recognize
`software_config.json`; those internal paths do not make it a required
customer input for the current module workflows.
