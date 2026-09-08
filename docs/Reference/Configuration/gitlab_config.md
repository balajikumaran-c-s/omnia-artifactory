# GitLab settings for Build Stream

GitLab settings are not stored in a standalone `gitlab_config.yml` in the
current Build Stream source. They are part of the consolidated
`build_stream_config.yml` at:

```text
$OMNIA_DATA_PATH/build_stream/input/project_default/build_stream_config.yml
```

Configure `gitlab_host`, `gitlab_project_name`,
`gitlab_project_visibility`, `gitlab_default_branch`,
`gitlab_https_port`, the `gitlab_min_*` resource checks,
`gitlab_puma_workers`, and `gitlab_sidekiq_concurrency` there.

See the [Build Stream configuration reference](build_stream_config.md) and
[Build Stream contract](../domain_contracts/build_stream_contract.md) for the
current schema and runtime paths.
