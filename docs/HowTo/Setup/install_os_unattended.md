# Unattended OS Installation via iDRAC Virtual Media

The unattended operating-system installation workflow is now owned by the
Utils domain. Use [Install an OS unattended](../utils/install_os_unattended.md)
for the current prerequisites, `install_os_config.yml` parameters, execution
commands, verification, and cleanup guidance.

The current workflow uses `src/utils/playbooks/install_os.yml` for both
`x86_64` and `aarch64`; the former `install_os_arm_node.yml` and
`iso_config.yml` procedure is obsolete.
