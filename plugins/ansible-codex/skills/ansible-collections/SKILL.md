---
name: ansible-collections
description: "Use when: choosing, installing, pinning, upgrading, or troubleshooting Ansible collections and roles with requirements.yml, ansible-galaxy, FQCN modules, adjacent collections, Automation Hub or Galaxy sources, version ranges, signatures, and Ansible 13 / ansible-core 2.20 porting compatibility."
argument-hint: "Describe needed modules, current requirements.yml, installed collections, Ansible version, and target platform."
---

# Ansible Collections

## Overview

Collections package Ansible modules, plugins, roles, and docs. Use `requirements.yml` to make dependencies reproducible and use FQCNs so generated playbooks point to the intended module. Current stable Ansible is version 13 on ansible-core 2.20.

## When to Use

- Selecting modules outside `ansible.builtin`
- Creating or updating `requirements.yml`
- Installing Galaxy or Automation Hub content
- Pinning collection versions for reproducible playbooks
- Fixing `couldn't resolve module/action` errors
- Preparing for Ansible major-version upgrades
- Managing collections adjacent to playbooks

## requirements.yml

```yaml
---
collections:
  - name: community.general
    version: ">=10.0.0,<12.0.0"
  - name: ansible.posix
    version: ">=1.6.0,<2.0.0"
  - name: community.docker
    version: ">=4.0.0,<5.0.0"

roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
```

Install both roles and collections from the same file:

```bash
ansible-galaxy install -r requirements.yml
```

Install only collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## FQCN Usage

Use FQCNs in playbooks and roles:

```yaml
- name: Configure firewall
  community.general.ufw:
    rule: allow
    port: '443'
    proto: tcp
```

Use `ansible-doc` to confirm module ownership and options:

```bash
ansible-doc community.general.ufw
ansible-doc ansible.builtin.systemd_service
```

## Version Strategy

| Situation | Version Approach |
|-----------|------------------|
| Application deployment repo | Pin a tested range such as `>=x.y,<next-major` |
| Regulated or offline environment | Pin exact versions and mirror artifacts |
| Shared role/collection development | Test against supported ranges in CI |
| Fast-moving lab | Allow wider ranges, but run lint and check mode before changes |

Ansible docs and ansible-lint recommend testing with newer tooling because it exposes upgrade problems earlier.

## Local Project Collections

For portable projects, install collections adjacent to playbooks:

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
```

Remember that Ansible expects the `ansible_collections/` path under configured collection paths. If using a custom path, configure it in `ansible.cfg`:

```ini
[defaults]
collections_path = ./collections:~/.ansible/collections
```

## Galaxy and Automation Hub Sources

Use the default Galaxy server for public community collections. For private content or Red Hat Automation Hub, configure server entries in `ansible.cfg` or use `--server`.

```ini
[galaxy]
server_list = automation_hub, galaxy

[galaxy_server.galaxy]
url = https://galaxy.ansible.com/
```

Do not put credentials in repository files. Use environment variables, CI secrets, netrc, or runner-level configuration.
Ansible 13 removed support for the Galaxy v2 server API. Any Galaxy-compatible server used for collections must support v3.

## Signature Awareness

If your organization requires signed collections, configure a GnuPG keyring and require valid signatures during install or verify. For unsigned community workflows, document that signature verification is not enforced.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `couldn't resolve module/action` | Collection not installed or FQCN typo | Install requirements and verify with `ansible-doc` |
| Module option not recognized | Collection version mismatch | Check `ansible-galaxy collection list` and pin compatible version |
| Works locally, fails in CI | CI did not install requirements | Add `ansible-galaxy install -r requirements.yml` |
| Deprecated module warning | Collection moved or module renamed | Update FQCN and requirements |
| Private Galaxy server stops working after upgrade | Server only supports the removed Galaxy v2 API | Move to a server that supports the Galaxy v3 API |
| Upgrade breaks templates or fact access | Ansible 13 retains stricter 2.19+ templating and deprecates injected facts | Run ansible-lint, switch new content to `ansible_facts[...]`, and review the porting guide |

## Upgrade Workflow

1. Read the target Ansible porting guide and collection changelogs.
2. Update version ranges in a branch.
3. Reinstall collections in a clean environment.
4. Run `ansible-lint .`.
5. Run syntax checks and check mode on limited hosts.
6. Fix FQCN, option, and templating issues.
7. Run a real limited deployment and a second idempotency run.

## Security Rules

- Do not embed credentials in Git collection URLs.
- Prefer SSH, netrc, extra headers, or CI-provided credentials for private repositories.
- Pin versions for production automation.
- Treat collection upgrades like code changes and validate before broad rollout.
