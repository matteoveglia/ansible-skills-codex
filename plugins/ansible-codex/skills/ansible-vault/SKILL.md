---
name: ansible-vault
description: "Use when: managing Ansible secrets, encrypting variables or files, choosing vault IDs, avoiding leaked passwords, adding no_log or diff false, using --ask-vault-pass, --vault-id, vault password files, CI secret handling, or migrating plaintext inventory secrets."
argument-hint: "Describe the secret type, environment count, current variable layout, and how playbooks will be run."
---

# Ansible Vault and Secrets

## Overview

Ansible Vault encrypts sensitive data such as passwords, tokens, private keys, and environment-specific credentials. Keep secret material out of plaintext inventory, logs, diffs, generated examples, and command output.

## When to Use

- Creating encrypted vars files
- Migrating plaintext secrets from inventory or group vars
- Adding vault IDs for dev/staging/production
- Preventing secrets from appearing in logs or `--diff`
- Designing CI-safe Ansible secret handling
- Debugging `Decryption failed` or missing vault password errors

## Recommended Layout

Use clear separation between public variables and vaulted values:

```
inventories/
└── production/
    └── group_vars/
        └── all/
            ├── vars.yml
            └── vault.yml
```

`vars.yml`:

```yaml
---
app_user: app
app_password: "{{ vault_app_password }}"
```

`vault.yml` is encrypted and contains:

```yaml
---
vault_app_password: change-me
```

## Core Commands

```bash
ansible-vault create inventories/production/group_vars/all/vault.yml
ansible-vault edit inventories/production/group_vars/all/vault.yml
ansible-vault view inventories/production/group_vars/all/vault.yml
ansible-vault encrypt inventories/production/group_vars/all/vault.yml
ansible-vault decrypt inventories/production/group_vars/all/vault.yml
ansible-vault rekey inventories/production/group_vars/all/vault.yml
```

Run playbooks with a prompt:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml --ask-vault-pass
```

Use vault IDs for multiple environments:

```bash
ansible-vault create --vault-id production@prompt inventories/production/group_vars/all/vault.yml
ansible-playbook playbooks/site.yml --vault-id production@prompt
```

In Ansible 13, the deprecated `vaultid` parameter was removed from the `vault` and `unvault` filters. Pick the Vault ID via `--vault-id`, configuration, or password sources instead of relying on filter arguments.

## CI and Automation

Prefer secret managers or CI-provided secret files over committing password files. If a vault password file is used, never commit it.

```bash
ansible-playbook playbooks/site.yml --vault-id production@/run/secrets/ansible-vault-production
```

A vault password file must be readable only by the runner user.

## Protecting Task Output

Vault protects files at rest, not values after they are decrypted during a play. Add output protections when a task can display secrets:

```yaml
- name: Render secret config
  ansible.builtin.template:
    src: secret.conf.j2
    dest: /etc/app/secret.conf
    owner: root
    group: root
    mode: '0600'
  no_log: true
  diff: false
  become: true
```

Use `no_log: true` on tasks that handle passwords, tokens, private keys, API payloads, or command output containing secrets.

## Migration Workflow

1. Search for likely secrets in vars, inventory, templates, and examples.
2. Move secret values into `vault.yml` with names prefixed by `vault_`.
3. Replace public variables with references to vaulted variables.
4. Encrypt the vault file.
5. Add `no_log: true` and `diff: false` to sensitive tasks.
6. Run `ansible-lint .` and a syntax check with the vault password available.
7. Verify no plaintext secret remains in git diff.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Decryption failed` | Wrong password or vault ID | Verify the ID and password source |
| `Attempting to decrypt but no vault secrets found` | Playbook references vaulted file without password | Add `--ask-vault-pass` or `--vault-id` |
| Secret appears in output | Decrypted value logged by a task | Add `no_log: true`; avoid debug output |
| Secret appears in diff | Template/copy diff enabled | Add `diff: false` |
| CI cannot decrypt | Secret file unavailable or wrong permissions | Mount/provide password securely at runtime |

## Safety Rules

- Do not commit vault password files.
- Do not paste real secrets into generated examples.
- Do not use `debug` on decrypted values.
- Do not rely on Vault alone for output secrecy.
- Prefer environment-specific vault IDs when teams or rotation schedules differ.
