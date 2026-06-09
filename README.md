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
ar_atomic_red_team_atomics: []
ar_atomic_red_team_atomic_files: []
ar_atomic_red_team_timeout_seconds: 60
ar_atomic_red_team_ansible_timeout_buffer_seconds: 30
```

- `ar_atomic_red_team_techniques`: A list of MITRE ATT&CK technique IDs to execute (e.g., `["T1003.001", "T1059.003"]`). Runs all atomics for each technique. Default is an empty list.
- `ar_atomic_red_team_atomics`: A list of specific atomics to execute. Each entry must include `technique` (MITRE ID) and `guid` (`auto_generated_guid` from the atomic YAML). Default is an empty list.
- `ar_atomic_red_team_atomic_files`: Custom atomic YAML files to deploy to the target host and execute. Each entry requires `path` (on the Ansible controller) or `content` (inline YAML). Optional: `technique`, `guid`, `src_dir` (companion `src/` folder for dependencies). Default is an empty list.
- `ar_atomic_red_team_timeout_seconds`: Maximum seconds for the atomic test execution step before `Invoke-AtomicTest` aborts. Applies to the main test run only (not prereqs or cleanup). Default is `60`.
- `ar_atomic_red_team_ansible_timeout_buffer_seconds`: Extra seconds added on top of `ar_atomic_red_team_timeout_seconds` for the Ansible `timeout` on the execution task (hard kill if the test process does not exit). Default is `30` (90s Ansible cap when the atomic timeout is 60s).

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

### Running Specific Atomics (technique + GUID)

Use `ar_atomic_red_team_atomics` when you need to run individual atomic tests. Provide the parent technique ID and the test `auto_generated_guid` from the atomic YAML.

```yaml
- hosts: windows_servers
  vars:
    ar_atomic_red_team_atomics:
      - technique: T1003.001
        guid: 0be2230c-9ab3-4ac2-8826-3199b9a0ebf8
      - technique: T1059.003
        guid: 9e8894c0-50bd-4525-a96c-d4ac78ece388
  roles:
    - ar_atomic_red_team
```

You can combine `ar_atomic_red_team_techniques` (all atomics per technique) and `ar_atomic_red_team_atomics` (single tests) in the same run.

### Running Custom Atomic YAML Files

Deploy and execute custom atomic YAML (for example, tests generated outside the stock Atomic Red Team repository):

```yaml
- hosts: windows_servers
  vars:
    ar_atomic_red_team_atomic_files:
      - path: /path/on/controller/T1003.001.yaml
      - path: /path/on/controller/custom.yaml
        guid: 0be2230c-9ab3-4ac2-8826-3199b9a0ebf8
      - content: |
          attack_technique: T9999
          display_name: Custom Test
          atomic_tests:
            - name: My custom test
              auto_generated_guid: 11111111-2222-3333-4444-555555555555
              description: Custom atomic for testing
              executor:
                name: command_prompt
                command: echo custom atomic
  roles:
    - ar_atomic_red_team
```

The role copies each file to `{atomics_root}/{technique}/{technique}.yaml` on the target host, then runs it via `Invoke-AtomicTest`. When `guid` is omitted and the YAML contains a single atomic test, that test's `auto_generated_guid` is used automatically.

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

For each entry in the execution queue (built from `ar_atomic_red_team_techniques`, `ar_atomic_red_team_atomics`, and/or `ar_atomic_red_team_atomic_files`):

1. **Linux**:
   - Runs `GetPrereqs` to install prerequisites
   - Executes the atomic test with `-TimeoutSeconds` and an Ansible task `timeout` of `timeout_seconds + buffer` (default 90s)
   - Runs `Cleanup` in an `always` block so cleanup still runs after timeouts or failures

2. **Windows**:
   - Runs `GetPrereqs` to install prerequisites
   - Executes the atomic test with `-TimeoutSeconds`, Ansible task `timeout`, and execution logging
   - Runs `Cleanup` in an `always` block so cleanup still runs after timeouts or failures

When a `guid` is provided on an entry, the role passes `-TestGuids` to `Invoke-AtomicTest` for that technique.

After all tests run, the role records per-atomic results in `ar_atomic_red_team_execution_results`
(technique, guid, success, return_code, stdout/stderr) and emits an **Atomic Red Team execution
summary** debug task for downstream consumers (Attack Range API, detection agent).

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

Each atomic test YAML includes an `auto_generated_guid` field. Pair it with the parent technique ID in `ar_atomic_red_team_atomics` to run a single test without executing every atomic for the technique.

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

## Integration with Attack Range

Attack Range passes playbook extra vars that map directly to this role:

| Attack Range (`simulate` extra var) | Role variable |
|-------------------------------------|---------------|
| `techniques` | `ar_atomic_red_team_techniques` |
| `atomics[].technique` + `atomics[].guid` | `ar_atomic_red_team_atomics` |
| `atomic_files[]` | `ar_atomic_red_team_atomic_files` |

Example API body (`POST /attack-range/simulate`):

```json
{
  "attack_range_id": "...",
  "target": "ar-win-1",
  "techniques": ["T1059.003"],
  "atomics": [
    {"technique": "T1003.001", "guid": "0be2230c-9ab3-4ac2-8826-3199b9a0ebf8"}
  ],
  "atomic_files": [
    {"path": "/path/on/controller/custom.yaml"},
    {"content": "attack_technique: T9999\n...", "guid": "11111111-2222-3333-4444-555555555555"}
  ]
}
```

The detection agent and Attack Range MCP use the same `techniques` / `atomics` / `atomic_files` shape.

## Related Projects

- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) - The underlying testing framework
- [MITRE ATT&CK](https://attack.mitre.org/) - The MITRE ATT&CK framework
- [Attack Range](https://github.com/splunk/attack_range) - Splunk's attack simulation platform
