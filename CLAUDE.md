# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this repo is

The **workloads layer** of a three-repo homelab on Proxmox VE: the app VMs,
one self-contained folder per app. Each app folder has its own `CLAUDE.md` —
read the one for the app you are working on (`nextcloud/CLAUDE.md`); this file
covers only what every app shares.

| Repo (siblings under `~/git-repos/`) | Owns |
|---|---|
| `homelab-proxmox` — core | Packer templates, `base-vm` module, `colac.homelab` collection, homelab-wide docs (`docs/ARCHITECTURE.md`) |
| `homelab-proxmox-monitoring` | The monitoring VM, and Elastic Agent enrollment on every host — this repo's VMs included |
| `homelab-proxmox-workloads` — **this** | The apps |

Changes to the template, the module, or `common`/`docker_data` belong in core
— propose them there, then bump the pin here. Anything about a VM's Elastic
Agent belongs in monitoring. [README.md](README.md) has the architecture and
the contracts; [TODO.md](TODO.md) is the single status tracker for this repo.

## Layout

```text
<app>/                      one per app: nextcloud/, k3s/
  README.md                 the app's runbook
  CLAUDE.md                 the app's deliberate decisions
  secrets.yaml(.example)    the app's own SOPS file (Ansible only)
  terraform/                one project = one VM = one TFC workspace (Local execution)
  ansible/                  ansible.cfg, inventory/, playbooks/, roles/,
                            requirements.yml, collections/ (git-ignored)
secrets.yaml(.example)      infrastructure credentials (Terraform only)
.mise/sops-exec             the only thing that decrypts; one profile per consumer
.mise/app                   runs a tool inside <app>/terraform or <app>/ansible
mise.toml                   tools, env, tasks (`mise tasks`)
```

## Decisions that are deliberate (do not "fix" these)

- **One folder per app, not one `terraform/` and one `ansible/` for all.** An
  agent or a human working on one app loads that app's context and that app's
  credentials, nothing else. Do not hoist an app's roles to the root; a role
  two apps need belongs in core's `colac.homelab` collection.
- **Secrets are split by consumer.** The root `secrets.yaml` holds only what
  Terraform needs; `<app>/secrets.yaml` only what that app's Ansible needs.
  `.mise/sops-exec` profiles enforce the split. Never add an app secret to the
  root file or a Proxmox credential to an app file.
- **Every Terraform project consumes core's `base-vm` at a pinned tag**
  (`?ref=vX.Y.Z`) and clones a core template by name (`template_name`). Never
  vendor the module or point `source` at a branch.
- **Each app's Ansible installs `colac.homelab` at a pinned tag into its own
  `collections/`** (`collections_path` in its `ansible.cfg`), so two apps — or
  this repo and monitoring — can sit on different core versions during a
  rollout.
- **No app enrolls its own Elastic Agent.** The monitoring repo lists the VM as
  an agent target. Do not add `elastic_agent` here.
- **Terraform never writes `group_vars/`**, and inventory host names differ
  from VM names (`nextcloud-vm` vs `nextcloud`) because a host and a group must
  not share a name.
- **The Docker data disk is Terraform's and Ansible's** (`data_disk_size` →
  `colac.homelab.docker_data`), never Packer's. A VM meant to *receive* an
  existing disk is created with `data_disk_size = null`.

## Toolchain & how to run things

mise pins every tool (`mise.toml`) and provides every entry point. There is no
Makefile and nothing is installed by hand.

```bash
mise run lint                 # pre-commit on all files + ansible-lint for every app
mise run lint:ansible         # just the Ansible half
mise run tf:validate          # fmt + validate every project, no backend, no credentials
mise run inventory nextcloud  # an app's inventory graph, no credentials
mise run secrets:check        # names missing keys in every file; never prints values
```

These need credentials and are for the human (ask first):
`mise run tf <app> plan|apply|…`, `mise run play <app> <playbook> [args]`.

- ansible-lint must pass at the **production** profile (each app's
  `ansible/.ansible-lint`).
- `colac.homelab` resolves only after `mise run setup` (or `deps:dev`, which
  installs the sibling core checkout instead of the pinned tag).
- Pins are load-bearing and shared with the other two repos: Terraform
  `1.15.7`, Telmate/proxmox `3.0.2-rc07`, ansible-core `2.17.14`,
  community.general `<13`.

## Secrets — SOPS + age, decrypted per command

- **Every `secrets.yaml` IS committed** — SOPS ciphertext. Never add one to
  `.gitignore`; `.gitleaks.toml` allowlists them.
- **Never read, print, `cat`, `grep` or `sed` any `secrets.yaml`,
  `mise.local.toml`, the age key, or any git-ignored `*.tfvars`. Never run
  `sops -d`, `sops decrypt` or `.mise/sops-exec` yourself.** Refer to secrets
  by key name; the `*.example` files list them, and `mise run secrets:check`
  reports missing ones without revealing values.
- **Nothing is exported into the shell.** `.mise/sops-exec <profile> <cmd>`
  decrypts one file for one command and exports only that profile's names,
  which `group_vars/` reads with `lookup('env', …)` — the only place they are
  consumed. Never reintroduce a literal secret into `group_vars`, a role
  default, or a rendered `.env`/`.ini`.
- The `while read` decrypt loop in `sops-exec` is deliberate: `eval` would
  re-parse plaintext as shell and break values with spaces, quotes or `$`.
- `nextcloud_users_json` is a single-line JSON array because
  `sops -d --output-type dotenv` emits flat `key=value` pairs only.

## Conventions

- **Conventional Commits** (commitlint + semantic-release). Never hand-edit
  `CHANGELOG.md` or bump versions. Commit only when asked; the human pushes.
- **Role variables are prefixed with the role name** (`nextcloud_aio_*`,
  `tailscale_*`, `reverse_proxy_*`), register names too. Non-secret shared
  values live in `inventory/group_vars/`, not role defaults.
- Markdown: heading levels increment by one (MD001); fenced code blocks
  declare a language (MD040).
