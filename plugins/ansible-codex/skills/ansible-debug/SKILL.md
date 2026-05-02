---
name: ansible-debug
description: "Use when: debugging Ansible failures, UNREACHABLE hosts, SSH authentication, become/sudo errors, MODULE FAILURE, undefined variables, template errors, Ansible 13 / ansible-core 2.20 upgrade issues, changed_when/failed_when problems, or slow playbooks."
argument-hint: "Paste the failing command, error output, inventory snippet, Ansible version, and target OS."
---

# Ansible Debugging

## Overview

Ansible errors usually fall into connection, authentication, privilege escalation, module, templating, syntax, or runtime behavior categories. Start by classifying the failure, then reduce the run to one host and one task.

## When to Use

- UNREACHABLE errors (SSH/network issues)
- Permission denied or sudo password errors
- MODULE FAILURE messages
- Undefined variable errors
- Template rendering failures
- Broken conditionals or runtime assumptions after Ansible 13 / ansible-core 2.20 upgrades
- Incorrect changed/failed reporting
- Slow playbook execution

## Error Categories

| Category | Symptoms | First Check |
|----------|----------|-------------|
| Connection | UNREACHABLE | `ssh -v user@host` |
| Authentication | Permission denied, Missing sudo password | SSH keys, sudoers config |
| Privilege escalation | become timeout, missing sudo password | `--ask-become-pass`, sudoers, `become` scope |
| Module | MODULE FAILURE | FQCN docs, parameters, target state |
| Templating | Undefined variable, non-boolean conditional, unsafe template | Variables, `when`, `failed_when`, Jinja syntax |
| Syntax | YAML parse error | Line number in error, indentation |

## Quick Diagnosis

### Connection Errors

```bash
ssh -v -i /path/to/key user@hostname
nc -zv hostname 22
ansible-inventory --host hostname
ansible hostname -m ansible.builtin.ping -vvv
```

Common causes:

- Wrong IP/hostname in inventory
- Firewall blocking port 22
- SSH key permissions other than `0600`
- Python missing or unusable on target
- Wrong `ansible_connection` for the platform

### Authentication Errors

```bash
ansible hostname -m ansible.builtin.ping -u user --private-key /path/to/key
ansible-playbook playbook.yml --ask-become-pass
```

For ansible-core 2.19+, the SSH connection plugin defaults to `SSH_ASKPASS` for SSH passwords instead of `sshpass`. If an environment depends on `sshpass`, set `ANSIBLE_SSH_PASSWORD_MECHANISM=sshpass` or configure `password_mechanism = sshpass` under `[ssh_connection]`.

### Privilege Escalation Errors

```bash
ansible hostname -m ansible.builtin.command -a 'whoami' --become -K -vvv
```

Checks:

- Is `become: true` needed for this task?
- Does the SSH user have sudo rights?
- Does sudo require a TTY or password?
- Is the failure actually `UNREACHABLE` after a become timeout? Use `ignore_unreachable`, not `ignore_errors`, only when continuing is intentional.

### Module Errors

```bash
ansible-doc ansible.builtin.copy
ansible-doc community.general.ufw
ansible-galaxy collection list
ansible --version
```

Check whether the FQCN is installed, the collection version matches the playbook, and the target has required Python libraries.

### Variable and Template Errors

```yaml
- name: Debug variable value
  ansible.builtin.debug:
    var: problematic_variable

- name: Validate required inputs
  ansible.builtin.assert:
    that:
      - app_port is defined
      - app_port | int > 0
```

For conditionals, do not use `{{ }}`. `when`, `failed_when`, `changed_when`, and `assert.that` are already Jinja expressions.

## Ansible 13 / ansible-core 2.20 Upgrade Failures

Before chasing playbook logic, confirm the runtime baseline:

- The controller running Ansible must use Python 3.12+.
- Managed nodes should provide Python 3.9+ for normal module execution.

Common upgrade-related failures:

| Error Pattern | Likely Cause | Fix |
|---------------|--------------|-----|
| `Conditionals must have a boolean result` | Condition returns a string/list/dict | Add explicit comparison or boolean test |
| `Template delimiters are not supported in expressions` | `{{ }}` inside `when`/`assert`/`failed_when` | Use the variable directly |
| `Type 'range' is unsupported for variable storage` | `range()` returned without conversion | Add `| list` or consume inline |
| Undefined value appears later than before | Lazy templating exposes nested undefineds | Access the correct nested path or set defaults |
| `result.exception` is missing after `failed_when: false` | ansible-core 2.20 renamed the field | Check `result.failed_when_suppressed_exception` instead |
| `ignore_files` type error in `include_vars` | A string was passed where a list is now required | Use `ignore_files: [".gitkeep"]` |
| Deprecation warnings for top-level facts | New content still uses injected fact names | Prefer `ansible_facts[...]` in new playbooks |

For deep template issues, run with traceback details:

```bash
ANSIBLE_DISPLAY_TRACEBACK=error ansible-playbook playbook.yml -vvv
```

## Verbosity Levels

| Flag | Shows |
|------|-------|
| `-v` | Task results |
| `-vv` | Task input parameters |
| `-vvv` | SSH connection details and many tracebacks |
| `-vvvv` | Full plugin internals |

Start with `-v`, increase only if needed.

## Debugging Commands

```bash
ansible-playbook --syntax-check playbook.yml
ansible-playbook --check playbook.yml
ansible-playbook --step playbook.yml
ansible-playbook --start-at-task "Task Name" playbook.yml
ansible-playbook --limit hostname playbook.yml
ansible-playbook playbook.yml --list-hosts
ansible-playbook playbook.yml --list-tasks
ansible-playbook playbook.yml --check --diff --limit hostname
```

## Common Error Patterns

| Error | Cause | Fix |
|-------|-------|-----|
| `Permission denied (publickey)` | SSH key not accepted | Check key permissions, key path, and `authorized_keys` |
| `Missing sudo password` | `become: true` without password or NOPASSWD | Use `--ask-become-pass` or configure sudoers |
| `No such file or directory` | Path does not exist | Create parent directories first |
| Package lock errors | Package manager locked | Wait for other process or remove stale lock only after verifying it is stale |
| `undefined variable` | Variable not defined or wrong nested path | Check spelling, source, and use `default()` where optional |
| `MODULE FAILURE` with little detail | Module exception or target dependency issue | Run with `-vvv`, check Python and module docs |
| Task always changed | Command/shell lacks idempotency | Use a module, `creates`/`removes`, or `changed_when` |
| Diff exposes secrets | `--diff` on sensitive template | Set `diff: false` and often `no_log: true` |

## Performance Debugging

```ini
[defaults]
callbacks_enabled = profile_tasks

[ssh_connection]
pipelining = True
```

```yaml
- hosts: all
  gather_facts: false
```

Use `gather_subset` for targeted facts, raise `forks` cautiously, and check whether slow tasks are waiting on package locks, service restarts, or serial strategy settings.
