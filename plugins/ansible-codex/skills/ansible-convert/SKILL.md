---
name: ansible-convert
description: "Use when: converting shell scripts, bash automation, manual runbooks, Dockerfile RUN steps, cloud-init snippets, or imperative server setup into idempotent Ansible playbooks, roles, handlers, templates, variables, and validation workflows."
argument-hint: "Paste the script or procedure and include target OSes, privilege model, desired inventory, and secrets constraints."
---

# Shell to Ansible Conversion

## Overview

Shell scripts execute commands imperatively; Ansible declares desired state. Conversion means rethinking operations as state declarations, not translating commands line-by-line. The goal is idempotency: running twice produces the same result without unintended changes.
Target the current stable baseline while converting: Ansible 13 runs on ansible-core 2.20 and keeps the stricter conditional behavior introduced in 2.19.

## When to Use

- Converting existing shell scripts to playbooks
- Migrating manual server setup procedures
- Replacing bash automation with Ansible
- Converting Dockerfile `RUN` commands or cloud-init snippets
- Turning one-off operational runbooks into repeatable roles

## Core Principle

Do not wrap shell commands in Ansible's `shell` module by default. Find the module that achieves the same end state declaratively.

```bash
mkdir -p /opt/app
chown app:app /opt/app
```

```yaml
- name: Ensure app directory exists
  ansible.builtin.file:
    path: /opt/app
    state: directory
    owner: app
    group: app
    mode: '0755'
```

## Conversion Table

| Shell Command | Ansible Module | Notes |
|---------------|----------------|-------|
| `mkdir -p` | `ansible.builtin.file` | `state: directory` |
| `cp` | `ansible.builtin.copy` | Static files |
| `cp` with variables | `ansible.builtin.template` | Use `.j2` templates |
| `rm -rf` | `ansible.builtin.file` | `state: absent` |
| `ln -s` | `ansible.builtin.file` | `state: link` |
| `chmod`, `chown` | Include in file/copy/template | `mode`, `owner`, `group` params |
| `apt-get install` | `ansible.builtin.apt` | `update_cache: true` |
| `dnf install` | `ansible.builtin.dnf` | Prefer on modern Fedora/RHEL family |
| `yum install` | `ansible.builtin.yum` | Legacy RHEL family; use `package` for generic tasks |
| `pip install` | `ansible.builtin.pip` | Specify `executable` if needed |
| `useradd` | `ansible.builtin.user` | Handles home, shell, groups |
| `groupadd` | `ansible.builtin.group` | Use before users if needed |
| `systemctl start` | `ansible.builtin.systemd_service` | `state: started` |
| `systemctl enable` | `ansible.builtin.systemd_service` | `enabled: true` |
| `systemctl daemon-reload` | `ansible.builtin.systemd_service` | `daemon_reload: true` |
| `curl -O` | `ansible.builtin.get_url` | Use `checksum` for verification |
| `curl` API call | `ansible.builtin.uri` | Use `status_code`, `body_format`, `headers` |
| `git clone` | `ansible.builtin.git` | Pin branch/tag/commit with `version` |
| `tar -xzf` | `ansible.builtin.unarchive` | `remote_src: true` if already on target |
| `echo >> file` | `ansible.builtin.lineinfile` | Ensures line exists |
| `cat > file` | `ansible.builtin.copy` | `content:` parameter |
| heredoc config | `ansible.builtin.template` | Move variable content to `.j2` |
| `sed -i` | `ansible.builtin.replace` or `lineinfile` | Prefer structured modules when available |
| `crontab` | `ansible.builtin.cron` | Declarative schedule entries |
| `iptables` | `ansible.builtin.iptables` or collection module | Consider firewalld/ufw modules by platform |
| apt source list + key | `ansible.builtin.deb822_repository` | Prefer signed repository definitions |

## Control Flow Conversion

### Conditionals

```bash
if [ -f /etc/debian_version ]; then
    apt-get install nginx
fi
```

```yaml
- name: Install nginx on Debian family hosts
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == "Debian"
```

Conditionals must evaluate to booleans in current-stable Ansible 13. Avoid truthy strings and avoid `{{ }}` in `when`, `failed_when`, and `changed_when`.

### Loops

```bash
for user in alice bob; do
    useradd $user
done
```

```yaml
- name: Ensure users exist
  ansible.builtin.user:
    name: "{{ item }}"
  loop:
    - alice
    - bob
```

Use dictionaries for multi-field loops:

```yaml
- name: Ensure users exist
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups | default(omit) }}"
  loop:
    - name: alice
      groups: sudo
    - name: bob
```

## Conversion Strategy

1. Identify phases: prerequisites, users, packages, files, services, validation, cleanup.
2. Extract hardcoded values into variables with safe defaults.
3. Replace each command with the narrowest idempotent module.
4. Use handlers for restarts triggered by file/template changes.
5. Use `block`/`rescue`/`always` only for real recovery or cleanup flows.
6. Add `no_log: true` and `diff: false` around secrets.
7. Validate with `ansible-lint`, syntax check, check mode, and a second real run.
8. Test on a controller with Python 3.12+ and, where possible, against managed nodes with Python 3.9+.

## Command and Shell Triage

Before using `command` or `shell`, ask:

- Does a module exist in `ansible.builtin` or an installed collection?
- Can `creates` or `removes` model idempotency?
- Can the command output be parsed into explicit `changed_when` and `failed_when`?
- Does it require shell features like pipes, redirection, globs, or environment expansion? If not, use `command` instead of `shell`.

## When Command or Shell is Necessary

Use `command` or `shell` only when no module exists. Always add proper change detection:

```yaml
- name: Run custom installer
  ansible.builtin.shell: /opt/app/install.sh
  args:
    creates: /opt/app/.installed
  register: install_result
  changed_when: "'Installed' in install_result.stdout"
  failed_when: install_result.rc != 0 and 'already installed' not in install_result.stderr
```

Prefer `command` unless shell features are required:

```yaml
- name: Run idempotent vendor CLI
  ansible.builtin.command:
    cmd: /usr/local/bin/vendorctl enable feature-a
    creates: /etc/vendor/feature-a.enabled
```

## Variable Extraction

Identify values to parameterize:

- Version numbers: `app_version: "1.2.3"`
- Paths: `app_dir: "/opt/app"`
- Usernames: `app_user: "appuser"`
- Ports: `app_port: 8080`

Use native YAML types for booleans and numbers:

```yaml
app_enabled: true
app_port: 8080
```

Place defaults in `defaults/main.yml` for roles. Do not store secrets in plaintext defaults. Use vaulted vars, external secret lookups, or environment-specific encrypted files.

## Dockerfile Conversion Notes

- `RUN apt-get update && apt-get install` becomes package tasks with cache updates.
- `COPY` becomes `copy` or `template` depending on variables.
- `ENV` becomes vars or service environment files.
- `USER` often becomes `become_user` or managed service user.
- `EXPOSE` often becomes firewall configuration or documentation; Ansible does not open ports by itself.
- `CMD`/`ENTRYPOINT` usually becomes a systemd unit, service config, or deployment documentation.

## Conversion Workflow

1. Read the whole script and identify major phases.
2. Map each command to an Ansible module.
3. Extract hardcoded values as variables.
4. Order tasks for dependencies, such as users before ownership and directories before files.
5. Add handlers for service restarts.
6. Add secret handling and `diff: false` where needed.
7. Run `ansible-lint .`.
8. Test with `--syntax-check`, then `--check --diff --limit one_host`.
9. Run for one host, then run again and verify `changed=0`.
