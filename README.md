# Ansible Codex Plugin

Reusable Ansible automation skills for Codex: playbook development, debugging, shell conversion, guided setup, linting, secrets, and collection management.

## What This Provides

This repository is a Codex plugin. It packages focused Ansible skills that Codex can invoke explicitly with `$` or select automatically when the task matches a skill description.

The plugin is intentionally instruction-only. It does not bundle MCP servers, app connectors, lab virtual machines, or generated project templates.

The repository root acts as a Codex marketplace. The installable plugin itself lives under `plugins/ansible-codex`.

## Installation

### From GitHub

```bash
codex plugin marketplace add matteoveglia/ansible-skills-codex
```

Then open Codex, run `/plugins`, choose the `Ansible Codex` marketplace, and install `ansible-codex`.

### Local Development

Clone the repository and add it as a local marketplace:

```bash
git clone https://github.com/matteoveglia/ansible-skills-codex.git
cd ansible-skills-codex
codex plugin marketplace add ./
```

Restart Codex, run `/plugins`, then install or enable `ansible-codex`.

### Verify Installation

In Codex, type `$` or run `/skills`. You should see skills like:

- `$ansible-playbook`
- `$ansible-debug`
- `$ansible-convert`
- `$ansible-interactive`
- `$ansible-lint`
- `$ansible-vault`
- `$ansible-collections`

## Skills Included

### ansible-playbook

Core playbook development reference. Use when creating playbooks, roles, or inventory files.

**Covers:**
- Project structure and ansible.cfg configuration
- Module patterns with FQCN (Fully Qualified Collection Names)
- Variable precedence rules
- Handlers and error handling
- Collection requirements and modern callback configuration
- Ansible 12 / ansible-core 2.19+ templating compatibility
- Common mistakes and fixes

### ansible-debug

Troubleshooting guide for Ansible errors. Use when playbooks fail with connection, authentication, or module errors.

**Covers:**
- Error category diagnosis (connection, auth, module, syntax)
- Verbosity levels and debugging commands
- Common error patterns and solutions
- Privilege escalation and unreachable-host handling
- Ansible 12 templating and conditional errors
- Performance debugging

### ansible-convert

Shell script to Ansible conversion. Use when migrating bash automation to idempotent playbooks.

**Covers:**
- Command-to-module mapping table
- Control flow conversion (conditionals, loops)
- When to use shell module
- Variable extraction patterns
- Dockerfile and runbook conversion notes
- Validation and idempotency workflow

### ansible-interactive

Step-by-step guided development. Use when starting a new Ansible project from scratch.

**Covers:**
- Environment analysis questions
- Project setup workflow
- Connectivity testing before playbooks
- Incremental development with validation
- Vault, requirements.yml, and safe first-run practices

### ansible-lint

Validation workflow for playbooks, roles, inventories, and CI. Use when checking generated or existing Ansible content before execution.

**Covers:**
- ansible-lint and syntax-check order
- Common rule fixes such as FQCN, no-changed-when, and risky file permissions
- Ansible 12 compatibility checks
- Minimal CI example

### ansible-vault

Secrets workflow for encrypted variables and safe task output. Use when adding credentials, migrating plaintext secrets, or designing CI decryption.

**Covers:**
- Vault file layout and naming conventions
- `--ask-vault-pass` and `--vault-id` workflows
- `no_log: true` and `diff: false`
- Secret migration and troubleshooting

### ansible-collections

Collection dependency workflow. Use when choosing modules, writing `requirements.yml`, pinning versions, or fixing module resolution errors.

**Covers:**
- Galaxy and Automation Hub dependency patterns
- FQCN usage and `ansible-doc` lookup
- Version pinning and adjacent collections
- Upgrade and porting workflow

## Usage Examples

**Create a playbook:**
```text
Create a playbook that installs nginx and configures it as a reverse proxy
```

**Convert a shell script:**
```text
Convert this deployment script to Ansible: [paste script]
```

**Debug an error:**
```text
My playbook fails with "UNREACHABLE" - help me debug
```

**Start from scratch:**
```text
Help me set up Ansible for my 5 Ubuntu servers step by step
```

**Validate content:**
```text
Run an Ansible validation audit on this role and fix the ansible-lint failures
```

**Handle secrets:**
```text
Move these plaintext inventory passwords into Ansible Vault and prevent them from appearing in diffs
```

**Manage dependencies:**
```text
Add the collections needed for these modules and pin them in requirements.yml
```

## Repository Structure

```
ansible-codex/
├── .agents/
│   └── plugins/
│       └── marketplace.json # Repo-local Codex marketplace
├── plugins/
│   └── ansible-codex/
│       ├── .codex-plugin/
│       │   └── plugin.json  # Codex plugin manifest
│       └── skills/
│           ├── ansible-playbook/
│           │   └── SKILL.md
│           ├── ansible-debug/
│           │   └── SKILL.md
│           ├── ansible-convert/
│           │   └── SKILL.md
│           ├── ansible-interactive/
│           │   └── SKILL.md
│           ├── ansible-lint/
│           │   └── SKILL.md
│           ├── ansible-vault/
│           │   └── SKILL.md
│           └── ansible-collections/
│               └── SKILL.md
├── README.md
└── LICENSE
```

## Codex Plugin Files

- `plugins/ansible-codex/.codex-plugin/plugin.json` is the plugin manifest Codex installs.
- `.agents/plugins/marketplace.json` exposes this repository as a Codex marketplace and points to `./plugins/ansible-codex`.
- `plugins/ansible-codex/skills/*/SKILL.md` contains the reusable skill instructions and trigger descriptions.

## Contributing

Contributions welcome. If you find patterns that work well in your Ansible projects, consider adding them to the skills. Open an issue or submit a PR.

## License

MIT
