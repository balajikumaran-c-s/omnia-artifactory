# Migration Guide

This guide helps you migrate from Omnia 2.2 to Omnia 2.3.

## Overview

Omnia 2.3 introduces a modular, capability-based deployment architecture that
reorganizes the documentation and execution model. This guide helps you
understand the changes and migrate your deployment.

## Key Changes in 2.3

### Modular deployment architecture

Omnia 2.3 organizes functionality into deployment modules. The names in code
formatting are their internal CLI identifiers:

- **repo_manager** - Repository management
- **image_build_manager** - Image building
- **discovery** - Node discovery
- **orchestrator** - Slurm, Kubernetes, networking, storage, authentication
- **telemetry** - Monitoring and metrics
- **build_stream** - BuildStreaM CI/CD
- **utils** - Utilities and helpers

Cross-module workflows are coordinated by Main; `main` is not a deployment
module.

### Execution Model Changes

**2.2**: Container-based execution
```bash
podman exec -it omnia_core bash
cd /omnia/<domain>
ansible-playbook playbook.yml
```

**2.3**: Module-based execution
```bash
./omnia.sh --run <domain> --tags <tag>
```

## Migration Steps

### Step 1: Update Configuration Files

Update your configuration files to use the new module-based structure:

- Move global settings to `omnia.env`
- Use module-specific config files (e.g., `repo_manager_config.yml`)
- Update configuration parameters to match new schema

### Step 2: Update Execution Commands

Replace container-based commands with module-based commands:

**Old (2.2)**:
```bash
podman exec -it omnia_core bash
cd /omnia/repo_manager
ansible-playbook repo.yml
```

**New (2.3)**:
```bash
./omnia.sh --run repo_manager --tags execute
```

### Step 3: Update Documentation References

Update any documentation references to use the new module-based structure:

- Update links from `HowTo/Setup/` to module-specific paths
- Update links from `HowTo/Slurm/` to `HowTo/orchestrator/`
- Update links from `HowTo/Kubernetes/` to `HowTo/orchestrator/`

### Step 4: Verify Deployment

After migration, verify your deployment:

```bash
# Check module status
./omnia.sh --status

# Validate configuration
./omnia.sh --validate

# Test module execution
./omnia.sh --run <domain> --tags validate
```

## Troubleshooting

If you encounter issues during migration:

1. Check the [Running Deployment Modules](../Overview/domain_execution.md) guide
2. Review the [Module Contracts](../Reference/domain_contracts/repo_manager_contract.md) for your module
3. Consult the [Troubleshooting](../Troubleshooting/index.md) section

## Related Documentation

- [Running Deployment Modules](../Overview/domain_execution.md)
- [Module Contracts](../Reference/domain_contracts/repo_manager_contract.md)
- [Getting Started: Full Deployment](full_deployment.md)


