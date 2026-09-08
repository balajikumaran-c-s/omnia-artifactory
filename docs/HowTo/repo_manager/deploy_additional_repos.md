# Publish Additional RPM Repositories

## Overview

The `additional_repos` section groups multiple upstream RPM repositories into
one aggregate Pulp distribution per architecture. Use it when downstream image
or provisioning workflows need a combined repository. For an independently
published custom repository, use `user_repos` instead.

Every `additional_repos` entry must resolve to the same effective priority.
When `priority` is omitted, its effective value is 99.

## Prerequisites

- Complete [Create Local Repositories](configure_repos.md).
- Ensure each upstream URL is reachable from the OIM and exposes valid RPM
  repository metadata.
- Identify packages in the catalog that reference each repository name.
- Choose one effective DNF priority for all entries in the architecture's
  `additional_repos` section.

## Procedure

1. Add the repositories under the target version and architecture:

    ~~~yaml
    repositories:
      "10.0":
        x86_64:
          additional_repos:
            grafana:
              url: "https://rpm.grafana.com/"
              gpgkey: "https://rpm.grafana.com/gpg.key"
              priority: 99
              policy: partial
              caching: true
    ~~~

2. Add catalog package sources whose `reponame` is exactly `grafana`, and make
   their groups reachable from the required functional layers. When adding
   more entries to `additional_repos`, keep their effective priorities equal.

3. Stage the configuration and run the normal reconciliation workflow:

    ~~~bash title="Run on: OIM host"
    cd <OMNIA_SOURCE_PATH>/src/repo_manager
    ./domain-init.sh
    cd playbooks
    ansible-playbook repo_manager.yml \
      --tags "precheck,download,status"
    ~~~

## Verification

List the RPM distributions and inspect the generated consumer output:

~~~bash title="Run on: OIM host"
pulp rpm distribution list --field name,base_path,base_url --limit 1000
sed -n '1,240p' \
  /opt/omnia/repo_manager/output/project_default/repo_status.yml
~~~

Confirm that the additional content is represented in the architecture's
repository output and that `overall_status` is `success`.

## Next steps

- [Build Cluster Images](../image_build_manager/build_images.md).
- [Make Additional Content Available to Image Builds](deploy_additional_packages.md).
- [Resynchronize Local RPM Repositories](../../Operations/repo_manager/local_repository_resync.md).

## Troubleshooting

- **Priorities conflict**: Give every entry in one `additional_repos` section
  the same explicit priority, or omit all priorities so they resolve to 99.
- **A repository is not synchronized**: Ensure a selected catalog package uses
  the exact repository key as its `reponame`.
- **A repository must remain independent**: Move it to `user_repos` and retain
  its catalog mapping.
- **An architecture is incomplete**: Define a separate `additional_repos` map
  under each catalog-selected architecture.
