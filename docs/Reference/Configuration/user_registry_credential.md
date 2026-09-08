# user_registry_credential.yml is retired

The current Repo Manager source does not consume the plaintext
`user_registry_credential.yml` format. Configure registry authentication in
`repo_manager_config.yml` with `auth.type: basic` and a
`credentials.vault_path`, then run the Repo Manager credential workflow.

The workflow writes encrypted credentials to
`repo_manager_config_credentials.yml` and stores its key in
`.repo_manager_config_credentials_key` in the Repo Manager project input
directory.

See [Repo Manager configuration](repo_manager_config.md).
