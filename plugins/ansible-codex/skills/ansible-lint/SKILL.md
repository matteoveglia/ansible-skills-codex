---
name: ansible-lint
description: "Use when: validating Ansible playbooks, roles, collections, tasks, inventories, or CI before execution. Covers ansible-lint, syntax checks, check mode, idempotency, FQCN, risky file permissions, no-changed-when, Ansible 12 compatibility, and remediation workflow."
argument-hint: "Describe the Ansible content to validate, lint output, CI failure, or target repository layout."
---

# Ansible Lint and Validation

## Overview

`ansible-lint` promotes proven Ansible patterns and catches content that may break during upgrades. Use it with the newest practical Ansible and ansible-lint versions, even when production controllers run older versions, because it helps surface future compatibility problems early.

## When to Use

- Reviewing generated playbooks, roles, or tasks
- Fixing `ansible-lint` CI failures
- Preparing content for Ansible 12 / ansible-core 2.19+ templating changes
- Checking FQCN usage, idempotency, risky permissions, and command/shell tasks
- Adding a validation workflow to a new Ansible project

## Validation Order

1. Install or update dependencies.
2. Install required collections.
3. Run `ansible-lint`.
4. Run syntax checks.
5. Run inventory checks.
6. Run check mode on a limited host.
7. Run once for real on a limited host.
8. Run again and confirm idempotency.

```bash
python -m pip install --upgrade ansible ansible-lint
ansible-galaxy collection install -r requirements.yml
ansible-lint .
ansible-playbook --syntax-check playbooks/site.yml
ansible-inventory -i inventories/staging/hosts.yml --list
ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --check --diff --limit host1
ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --limit host1
ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --limit host1
```

The final run should report `changed=0` unless the playbook intentionally changes every run.

## Common Rule Fixes

| Rule Area | What It Usually Means | Preferred Fix |
|-----------|------------------------|---------------|
| FQCN | Short module name used | Use `ansible.builtin.copy` or collection FQCN |
| `command-instead-of-module` | A module exists for the operation | Replace command with the module |
| `no-changed-when` | Command/shell always reports changed | Add `changed_when`, `creates`, or `removes` |
| `risky-file-permissions` | File task can create unsafe permissions | Set explicit quoted mode like `'0644'` |
| `package-latest` | Package updates are unpinned/uncontrolled | Use `state: present` unless latest is intentional |
| `yaml` | Formatting or truthy value issue | Use native booleans and consistent YAML style |
| `no-log-password` | Secret-like value may be logged | Add `no_log: true` and avoid printing secrets |

## Minimal .ansible-lint

Only configure what the project genuinely needs. Avoid broad skips.

```yaml
---
profile: production
exclude_paths:
  - .cache/
  - collections/
warn_list:
  - experimental
```

If a rule must be skipped, prefer a narrow inline or file-level justification over disabling it globally.

## Ansible 12 Compatibility Checks

Pay special attention to conditionals and templating:

```yaml
# Good
when: app_enabled | bool
failed_when: command_result.rc != 0 or 'ERROR' in command_result.stderr
changed_when: command_result.rc == 0 and 'changed' in command_result.stdout

# Avoid
when: "{{ app_enabled }}"
failed_when: "{{ command_result.rc != 0 }}"
```

Conditionals must return booleans. Do not rely on non-empty strings, dicts, or lists being truthy.

## CI Baseline

Use this as a simple CI step:

```yaml
---
name: Ansible validation
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - run: python -m pip install --upgrade ansible ansible-lint
      - run: ansible-galaxy collection install -r requirements.yml
        if: hashFiles('requirements.yml') != ''
      - run: ansible-lint .
```

## Remediation Workflow

1. Fix syntax and YAML errors first; later checks depend on parseable content.
2. Install missing collections rather than rewriting valid FQCNs away.
3. Replace commands with modules where possible.
4. Add explicit idempotency to remaining commands.
5. Add explicit file modes.
6. Protect secrets with `no_log: true`, `diff: false`, and Vault or external secret lookups.
7. Re-run lint and targeted playbook validation.
