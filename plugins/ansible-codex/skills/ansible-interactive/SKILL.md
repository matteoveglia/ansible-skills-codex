---
name: ansible-interactive
description: "Use when: guiding step-by-step Ansible setup, bootstrapping a new project, interviewing for inventory details, teaching Ansible hands-on, validating connectivity, creating ansible.cfg, inventories, requirements.yml, first playbooks, and safe run workflows."
argument-hint: "Describe the servers, OSes, SSH user/key, sudo model, environments, and automation goal."
---

# Interactive Ansible Development

## Overview

Interactive development builds automation incrementally with continuous validation. Gather connection facts first, prove inventory works, then add one behavior at a time. This catches errors while they are still small.

## When to Use

- Setting up Ansible for a new environment
- Teaching someone Ansible hands-on
- Building playbooks incrementally with validation
- Troubleshooting connectivity before automation

## Development Phases

### Phase 1: Environment Analysis

Gather before writing any code:

| Question | Why It Matters |
|----------|----------------|
| How many servers? | Affects inventory organization |
| IP addresses/hostnames? | Required for inventory |
| SSH user and key location? | Connection configuration |
| Password or key auth? | Determines SSH setup |
| Sudo with or without password? | Privilege escalation config |
| Server roles (web, db, app)? | Inventory grouping |
| Operating systems? | Module selection |
| Secrets involved? | Vault or external secret workflow |
| Collections needed? | `requirements.yml` and version pinning |
| Environments? | Inventory directory layout |

Verify Ansible is installed:

```bash
ansible --version
ansible-galaxy collection list
```

### Phase 2: Project Setup

Create minimal structure:

```bash
mkdir ansible-project && cd ansible-project
mkdir -p inventories/staging/group_vars/all inventories/staging/host_vars playbooks roles
```

**ansible.cfg:**

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

Keep `host_key_checking` enabled by default. Disable it only for short-lived lab hosts where trust-on-first-use is intentionally not required.

**requirements.yml:**

```yaml
---
collections: []
```

Add collections here as soon as a playbook uses non-`ansible.builtin` modules.

**inventories/staging/hosts.yml:**

```yaml
---
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.10
          ansible_user: admin
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
    dbservers:
      hosts:
        db1:
          ansible_host: 192.168.1.20
          ansible_user: admin
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

### Phase 3: Connectivity Test

Always test before writing playbooks:

```bash
ansible-inventory --list
ansible all -m ansible.builtin.ping
```

| Result | Action |
|--------|--------|
| SUCCESS | Proceed to playbooks |
| UNREACHABLE | Check `ssh -v user@host` |
| Permission denied | Verify key path, permissions, user, and authorized key |
| Sudo password required | Add `--ask-become-pass` or configure NOPASSWD |

If privilege escalation is required, test it separately:

```bash
ansible all -m ansible.builtin.command -a 'whoami' --become -K
```

### Phase 4: Incremental Playbook Development

Start simple, add one task at a time:

```yaml
---
- name: Baseline host facts
  hosts: all
  tasks:
    - name: Show OS info
      ansible.builtin.debug:
        msg: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
```

Run:

```bash
ansible-playbook playbooks/site.yml
```

Then add tasks one by one, testing after each:

```yaml
    - name: Ensure nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present
      become: true
```

When a task changes a service config, introduce the handler immediately:

```yaml
    - name: Render nginx config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      notify: Reload nginx
      become: true

  handlers:
    - name: Reload nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: reloaded
      become: true
```

### Phase 5: Validation Cycle

After each change:

1. `ansible-lint .`
2. `ansible-playbook --syntax-check playbooks/site.yml`
3. `ansible-playbook playbooks/site.yml --check --diff --limit one_host`
4. `ansible-playbook playbooks/site.yml --limit one_host`
5. Run again and verify `changed=0` unless the playbook intentionally changes state

Use `--list-hosts`, `--list-tasks`, and `--tags` to explain scope before touching systems.

### Phase 6: Secrets

If credentials are needed:

```bash
ansible-vault create inventories/staging/group_vars/all/vault.yml
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Keep secret variable names clearly distinct, such as `vault_app_password`, then reference them from non-secret vars if needed. Add `no_log: true` and `diff: false` to tasks that might print secrets.

## Red Flags - Stop and Debug

- Adding multiple untested tasks at once
- Skipping `--check` before real runs
- Ignoring `changed` on second run
- Not testing SSH before writing playbooks
- Hiding broad failures with `ignore_errors`
- Putting secrets in plaintext inventory
- Using `{{ }}` inside `when`, `failed_when`, `changed_when`, or `assert.that`

## Communication Pattern

When guiding users:

- Explain what will happen before running commands
- After completion, summarize what was done
- When multiple approaches exist, present options with tradeoffs
- Acknowledge progress at milestones
- Ask before running commands that change remote systems
