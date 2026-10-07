# AGENTS.md — homelab-proxmox-workloads

Instructions for AI coding agents working in this repo. Humans start at
[README.md](README.md). Each app folder has its own `AGENTS.md` (loaded via
its `CLAUDE.md` when you work there) — read it before changing that app.

## What this repo is

The workloads layer of a three-repo Proxmox homelab: the app VMs, one
self-contained folder per app (`nextcloud/` live, `k3s/` a Terraform recipe
with no VM deployed). Siblings in `~/git-repos/`:

- `homelab-proxmox` — core: templates, `base-vm`, the `colac.homelab`
  collection, and the homelab-wide docs
- `homelab-proxmox-monitoring` — the Elastic Stack, and the Elastic Agent on
  every VM, this repo's included

Template, module, or `common`/`docker_data` changes belong in core — propose
them there, then bump the pin here. Anything about a VM's Elastic Agent belongs
in monitoring.

## Where to look

- [README.md](README.md) — apps, architecture, contracts, adding an app
- `<app>/README.md` — that app's runbook; `<app>/AGENTS.md` — its decisions
- Core's docs (`../homelab-proxmox/docs/`): `CREDENTIALS.md` (issue, store,
  rotate every credential), `DEVELOPMENT.md` (tasks, conventions, releases),
  `ARCHITECTURE.md`
- [TODO.md](TODO.md) — status; check items off there, never in this file

## Commands

```bash
mise run lint                 # pre-commit on all files + ansible-lint for every app
mise run tf:validate          # every app's Terraform, no backend, no credentials
mise run inventory <app>      # no credentials
mise run secrets:check        # every secrets file, missing key names only
```

Need credentials — ask the human first: `mise run tf <app> <args>`,
`mise run play <app> <playbook> [args]`. The app always comes first.

## Rules

- **Secrets:** never read, print, `cat`, `grep` or `sed` any `secrets.yaml`,
  `mise.local.toml`, the age key, or `*.tfvars`; never run `sops -d`,
  `sops decrypt` or `.mise/sops-exec`. Refer to secrets by key name (the
  `*.example` files) and use `mise run secrets:check`. New secret → core's
  `docs/CREDENTIALS.md` § Adding a secret.
- **Secrets are split by consumer.** The root `secrets.yaml` holds only what
  Terraform needs; `<app>/secrets.yaml` only that app's Ansible secrets. Never
  put an app secret in the root file or a Proxmox credential in an app file.
- **Git:** Conventional Commits; never edit `CHANGELOG.md` or version numbers;
  commit only when asked; the human pushes.
- **Ansible:** ansible-lint stays green at the production profile; role
  variables and `register:` names carry the role prefix.

## Deliberate decisions — do not "fix"

- **One folder per app.** Do not hoist an app's roles or inventory to the
  root; a role two apps need goes into core's `colac.homelab`.
- **Core is consumed at a pinned tag** — `?ref=vX.Y.Z` for `base-vm`,
  `version: vX.Y.Z` in each `requirements.yml`, moved together. Never vendor
  the module or point a source at a branch.
- **Each app installs its collections into its own `ansible/collections/`**,
  so apps can sit on different core versions during a rollout.
- **No app enrolls its own Elastic Agent.** Monitoring lists the VM as an
  agent target. Do not add `elastic_agent` here.
- **Inventory host names differ from VM names** (`nextcloud-vm` vs
  `nextcloud`) — a host and a group must not share a name. Terraform never
  writes `group_vars/`.
- **A VM meant to receive an existing Docker data disk** is created with
  `data_disk_size = null`.
