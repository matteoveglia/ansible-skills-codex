---
name: ansible-playbook
description: "Use when: creating Ansible playbooks, roles, tasks, handlers, inventories, group_vars, host_vars, ansible.cfg, or reusable automation. Covers idempotency, FQCN modules, collections, variables, handlers, check mode, Ansible 13 / ansible-core 2.20 guidance, stricter 2.19+ templating behavior, and validation workflows."
argument-hint: "Describe the infrastructure task, target OSes, inventory shape, and desired playbook or role output."
---

# Ansible Playbook Development

## Overview

Ansible playbooks declare desired system state rather than imperative command sequences. Optimize for idempotency, readable task names, explicit inputs, and validation before execution. Prefer built-in or collection modules over `command`, `shell`, or `raw`.

## When to Use

- Creating new playbooks or roles
- Writing inventory files
- Designing reusable `group_vars`, `host_vars`, defaults, handlers, and templates
- Choosing modules and collections
- Reviewing YAML, conditionals, variables, and handler behavior
- Understanding variable precedence
- Preparing playbooks for current-stable Ansible 13 / ansible-core 2.20 behavior

## Project Structure

```
project/
├── ansible.cfg
├── requirements.yml
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   └── host_vars/
│   └── staging/
├── playbooks/
│   └── site.yml
└── roles/
    └── app/
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── tasks/main.yml
        ├── templates/
        └── meta/main.yml
```

Use a single top-level inventory for small projects. Use `inventories/<env>/` when environments need separate variables or dynamic inventory config.

## Essential ansible.cfg

```ini
[defaults]
inventory = ./inventories/staging/hosts.yml
roles_path = ./roles
stdout_callback = ansible.builtin.default
callback_result_format = yaml
show_task_path_on_failure = True
retry_files_enabled = False

[privilege_escalation]
become = False
become_method = sudo
```

Set `become: true` at the play or task level when elevated privileges are actually required. Avoid disabling `host_key_checking` except for disposable labs.
Do not set `transport = smart`; the deprecated `smart` transport selector was removed in Ansible 13.

## Collection Requirements

Pin non-core collections so every controller uses the same modules:

```yaml
---
collections:
  - name: community.general
    version: ">=10.0.0,<12.0.0"
  - name: ansible.posix
    version: ">=1.6.0,<2.0.0"
```

Install with:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Module Patterns

| Operation | Module | Key Parameters |
|-----------|--------|----------------|
| Create directory | `ansible.builtin.file` | `state: directory`, `mode`, `owner` |
| Copy file | `ansible.builtin.copy` | `src`, `dest`, `mode` |
| Template | `ansible.builtin.template` | `src`, `dest`, variables in `.j2` |
| Install package | `ansible.builtin.package` | `name`, `state: present` |
| Manage systemd unit | `ansible.builtin.systemd_service` | `name`, `state`, `enabled`, `daemon_reload` |
| Manage generic service | `ansible.builtin.service` | `name`, `state`, `enabled` |
| APT repository | `ansible.builtin.deb822_repository` | `name`, `uris`, `suites`, `components`, `signed_by` |
| Wait for connection | `ansible.builtin.wait_for_connection` | `timeout`, `connect_timeout` |
| Validate condition | `ansible.builtin.assert` | `that`, `fail_msg`, `success_msg` |
| Run command | `ansible.builtin.command` | `cmd`, `creates`/`removes`, register result, `changed_when` |

Use FQCNs (`ansible.builtin.copy`, `community.general.ufw`) in generated content. They make docs lookup precise and avoid name collisions.

## Variable Precedence

Lowest to highest for common playbook work:

1. Role defaults (`defaults/main.yml`)
2. Inventory group vars
3. Inventory host vars
4. Playbook vars
5. Role vars (`vars/main.yml`)
6. Task vars
7. Extra vars (`-e`)

Avoid role `vars/main.yml` for values users should override. Prefer role defaults and document expected variables in `README.md` or `meta/argument_specs.yml` for roles.

## Ansible 13 / ansible-core 2.20 Compatibility

Ansible 13 is based on ansible-core 2.20. Generate new content with the current stable baseline in mind:

- Controller Python must be 3.12+.
- Managed nodes generally need Python 3.9+ for normal module execution.
- Prefer `ansible_facts[...]` in new content. `INJECT_FACTS_AS_VARS` is deprecated and will change default behavior in ansible-core 2.24.
- `include_vars.ignore_files` must be a list, not a string.
- If `failed_when` suppresses an error, the result key is now `failed_when_suppressed_exception`.
- Modern Ansible is stricter about templates and conditionals. Generate content with these rules:

- `when`, `failed_when`, `changed_when`, and `assert.that` are raw Jinja expressions. Do not wrap variables in `{{ }}` there.
- Conditionals must produce booleans, not truthy strings, lists, or dictionaries.
- Convert types explicitly with filters like `| bool`, `| int`, `| string`, or `| list` when needed.
- Use `default(omit)` when an optional module argument should be omitted.
- Do not construct dynamic expressions with nested templates. Refactor to explicit variables or separate tasks.

Good:

```yaml
- ansible.builtin.assert:
    that:
      - inventory_hostname | length > 0
      - app_port | int > 0
```

Avoid:

```yaml
- ansible.builtin.assert:
    that:
      - inventory_hostname
      - "{{ app_port }} > 0"
```

## Handlers

```yaml
tasks:
  - name: Update config
    ansible.builtin.template:
      src: app.conf.j2
      dest: /etc/app.conf
      mode: '0644'
    notify: Restart app

handlers:
  - name: Restart app
    ansible.builtin.systemd_service:
      name: app
      state: restarted
```

Use `force_handlers: true` when a changed configuration must always trigger its restart even if a later task fails, as long as the host remains reachable.

## Error Handling

```yaml
- block:
    - name: Risky operation
      ansible.builtin.command: /opt/app/upgrade.sh
      register: upgrade_result
      changed_when: "'upgraded' in upgrade_result.stdout"
  rescue:
    - name: Handle failure
      ansible.builtin.debug:
        msg: "Upgrade failed, rolling back"
  always:
    - name: Cleanup
      ansible.builtin.file:
        path: /tmp/upgrade.lock
        state: absent
```

Use `ignore_unreachable` for unreachable hosts; `ignore_errors` does not handle connection failures, undefined variables, missing packages, or syntax errors.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using short module names | Use FQCN: `ansible.builtin.copy` not `copy` |
| Hardcoded values | Extract to variables in `defaults/main.yml` |
| Missing `changed_when` on commands | Add explicit change detection or use module idempotency |
| Forgetting handler flush | Use `meta: flush_handlers` when needed before dependent tasks |
| YAML indentation errors | Use 2 spaces, never tabs |
| Colon in unquoted string | Quote values containing `: ` |
| Secrets in vars or templates | Use `ansible-vault`, external secret lookups, `no_log: true`, and `diff: false` |
| Global `become = True` | Set privilege escalation only where needed |
| `stdout_callback = yaml` | Use `ansible.builtin.default` with `callback_result_format = yaml` |
| Top-level injected facts in new content | Prefer `ansible_facts['distribution']`-style access |
| `transport = smart` | Remove it; Ansible 13 no longer supports the `smart` transport selector |
| Truthy string conditionals | Make conditions explicit booleans |

## Verification Commands

```bash
ansible-lint .
ansible-playbook --syntax-check playbooks/site.yml
ansible-inventory -i inventories/staging/hosts.yml --list
ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --check --diff --limit host1
ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --limit host1
ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --limit host1
```

The second real run should report `changed=0` unless the playbook intentionally manages changing state.
