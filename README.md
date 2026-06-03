# Ansible Role: Atomic Red Team

This role installs and runs [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) tests on Linux and Windows systems. The role automatically checks if Atomic Red Team is already installed and only installs it if needed, making it idempotent and safe to run multiple times.

## Overview

Atomic Red Team is a library of tests mapped to the MITRE ATT&CK framework. This Ansible role:

- **Automatically installs** Atomic Red Team if not already present (supports both Linux and Windows)
- **Runs Atomic Red Team techniques** specified in the configuration
- **Handles prerequisites** and cleanup automatically for each technique
- **Supports multiple platforms**: Ubuntu (Linux) and Windows

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
ar_atomic_red_team_techniques: []
ar_atomic_red_team_guids: []
```

- `ar_atomic_red_team_techniques`: A list of MITRE ATT&CK technique IDs to execute (e.g., `["T1003.001", "T1059.003"]`). Runs all atomics for each technique. Default is an empty list.
- `ar_atomic_red_team_guids`: A list of atomic test `auto_generated_guid` values to execute (e.g., `["5c2571d0-1572-416d-9676-812e64ca9f44"]`). Runs only the matching atomic test. The role resolves each GUID to its technique ID before execution. Default is an empty list.

## Dependencies

None.

## Requirements

- **For Linux (Ubuntu)**:
  - PowerShell Core (`pwsh`) must be installed
  - System must be running Ubuntu (detected via `ansible_distribution`)
  - Requires `become: true` privileges

- **For Windows**:
  - PowerShell 5.1 or later
  - System must be running Windows (detected via `ansible_distribution`)
  - Requires WinRM connectivity

## Example Playbook

### Running on Ubuntu

```yaml
- hosts: ubuntu_servers
  become: true
  vars:
    ar_atomic_red_team_techniques:
      - T1003.001  # OS Credential Dumping: LSASS Memory
      - T1059.003  # Command and Scripting Interpreter: Windows Command Shell
  roles:
    - ar_atomic_red_team
```

### Running on Windows

```yaml
- hosts: windows_servers
  vars:
    ar_atomic_red_team_techniques:
      - T1003.001
      - T1059.003
  roles:
    - ar_atomic_red_team
```

### Running a Specific Atomic by GUID

Use `ar_atomic_red_team_guids` when you need to run exactly one atomic test. GUIDs are stable identifiers from the atomic test YAML (`auto_generated_guid` field).

```yaml
- hosts: windows_servers
  vars:
    ar_atomic_red_team_guids:
      - 5c2571d0-1572-416d-9676-812e64ca9f44
  roles:
    - ar_atomic_red_team
```

You can combine technique IDs and GUIDs in the same playbook run. Technique IDs run all atomics for that technique; GUIDs run only the matching test.

### Running on Mixed Environments

```yaml
- hosts: all
  become: true
  vars:
    ar_atomic_red_team_techniques:
      - T1003.001
      - T1059.003
  roles:
    - ar_atomic_red_team
```

The role automatically detects the operating system and applies the appropriate installation and execution methods.

## What This Role Does

### Installation Phase

The role performs the following checks and installations:

1. **Linux (Ubuntu)**:
   - Checks if `/root/AtomicRedTeam/atomics` directory exists
   - Checks if `/root/AtomicRedTeam/invoke-atomicredteam/Invoke-AtomicRedTeam.psd1` module exists
   - Installs Atomic Red Team if either component is missing
   - Creates PowerShell profile directory and configuration

2. **Windows**:
   - Checks if `C:\AtomicRedTeam\atomics` directory exists
   - Checks if `C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1` module exists
   - Installs Atomic Red Team if either component is missing
   - Configures .NET strong cryptography settings
   - Installs NuGet provider and PowerShell-YAML module
   - Sets Internet Explorer first-run customization

### Execution Phase

For each entry in the execution queue (built from `ar_atomic_red_team_techniques` and/or resolved `ar_atomic_red_team_guids`):

1. **Linux**:
   - Runs `GetPrereqs` to install prerequisites
   - Executes the atomic test (all tests for a technique ID, or a single test when a GUID is specified via `-TestGuids`)
   - Runs `Cleanup` to remove artifacts

2. **Windows**:
   - Runs `GetPrereqs` to install prerequisites
   - Executes the atomic test with timeout and logging
   - Runs `Cleanup` to remove artifacts

When a GUID is provided, the role scans the installed atomics folder to resolve the parent technique ID, then passes `-TestGuids` to `Invoke-AtomicTest`.

## Idempotency

This role is **idempotent** - it checks for existing Atomic Red Team installations before attempting to install. This means:

- Safe to run multiple times
- Won't reinstall if already present
- Only installs missing components
- Reduces execution time on subsequent runs

## Finding Technique IDs and Atomic GUIDs

MITRE ATT&CK technique IDs follow the pattern `T####` or `T####.###` (e.g., `T1003.001`). You can find techniques at:

- [MITRE ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/)
- [Atomic Red Team GitHub](https://github.com/redcanaryco/atomic-red-team)
- [MITRE ATT&CK Website](https://attack.mitre.org/)

Each atomic test YAML includes an `auto_generated_guid` field. Use that value with `ar_atomic_red_team_guids` to run a single test without executing every atomic for the technique.

## Customization

### Running Specific Techniques

```yaml
vars:
  ar_atomic_red_team_techniques:
    - T1003.001  # OS Credential Dumping: LSASS Memory
    - T1055      # Process Injection
    - T1059.003  # Command and Scripting Interpreter: Windows Command Shell
```

### Empty List (Installation Only)

If you only want to install Atomic Red Team without running any tests:

```yaml
vars:
  ar_atomic_red_team_techniques: []
```

## Troubleshooting

### Linux Issues

- **PowerShell not found**: Ensure PowerShell Core is installed (`pwsh` command available)
- **Permission errors**: Ensure `become: true` is set in your playbook
- **Installation fails**: Check internet connectivity and GitHub access

### Windows Issues

- **WinRM connectivity**: Ensure WinRM is properly configured
- **Execution policy**: The role sets execution policy to bypass automatically
- **Module installation fails**: Check internet connectivity and PowerShell gallery access

## License

Apache License 2.0

## Author Information

This role was created by [P4T12ICK](https://github.com/P4T12ICK)

## Related Projects

- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) - The underlying testing framework
- [MITRE ATT&CK](https://attack.mitre.org/) - The MITRE ATT&CK framework
- [Attack Range](https://github.com/splunk/attack_range) - Splunk's attack simulation platform
