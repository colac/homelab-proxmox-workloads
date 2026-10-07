# TODO / Roadmap

The single place this repo's status lives. When something lands, check it off
here — `README.md` and `AGENTS.md` point at this file rather than duplicating
status inline. Template and homelab-wide items are tracked in the core repo's
`TODO.md`; monitoring in that repo's.

## Nextcloud

- [x] VM from the core 24.04 template → TFC workspace `Nextcloud`
- [x] Nextcloud AIO in reverse-proxy mode behind Caddy, Let's Encrypt via
      Cloudflare DNS-01
- [x] TrueNAS SMB external storage: per-user private folders plus a read-only
      `Familia` mount
- [x] Tailscale subnet route + PiHole split-DNS for off-LAN reach
- [ ] Verify Nextcloud's own Borgbackup covers the database and app config,
      and document the restore path in `nextcloud/README.md` — the SMB
      external storage is explicitly **not** covered by it
- [ ] **Its own Cloudflare token.** The split starts with the one shared
      token copied into both repos' secrets. Mint a second (Zone:DNS:Edit, same
      zone) for one of them, confirm both certs renew, and only then are they
      independent
- [ ] **Move to the 26.04 template and a data disk** — a `template_name`
      change and a re-clone, so it needs the AIO backup/restore path above
      first

## k3s

- [x] Terraform recipe on the core 26.04 template → TFC workspace `k3s`
      (24G OS + 12G data)
- [ ] **Sync the state with reality.** The VM was deleted from Proxmox, but
      the workspace state may still list it, so plans propose creating it:
      `mise run tf k3s apply -refresh-only`, or `apply` when it is wanted again
- [ ] `k3s/ansible/` — nothing configures the VM yet
- [ ] Add it to the monitoring repo's agent targets once it runs something

## Repo / tooling

- [x] **Split out of the homelab monorepo.** History for these paths kept via
      `git filter-repo`; one folder per app; secrets split into infra (root)
      and per-app files; the shared roles and module come from core at a
      pinned tag; mise replaces the Makefile and direnv
- [x] **On core v2.0.1, Nextcloud plan clean.** Applied once to move the
      VM's CPU from the deprecated `cores` into the `cpu { }` block (same 4
      cores, `host` type — no reboot); `startup_shutdown` drift gone. Plans
      say "No changes"
- [x] **Docs:** `AGENTS.md` (+ `nextcloud/AGENTS.md`) imported by one-line
      `CLAUDE.md` files; credentials and development docs centralised in
      core's `docs/`; `mise run deps:dev` also points Terraform at the
      sibling core through a git-ignored `dev_override.tf`
- [ ] **A dedicated Proxmox token for this repo**
      (`terraform@pve!terraform-workloads`) so revoking the monitoring one
      cannot break this one
- [ ] **A read-only agent identity.** A separate age key and a
      `PVEAuditor`-only token an AI agent could use for `plan`, with `apply`
      and `play` kept for the human
- [ ] **Enable the commented-out pre-commit hooks** (`ansible-lint`,
      `yamllint`, `gitleaks`)
