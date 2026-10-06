# Ansible — the Nextcloud VM

Configures the VM `nextcloud/terraform` provisions: Nextcloud All-in-One
behind a **Caddy** reverse proxy that terminates TLS for a real domain (Let's
Encrypt via Cloudflare DNS-01), reachable privately over the LAN and, off-LAN,
over a **Tailscale subnet route**, with TrueNAS attached as SMB external
storage. The end-to-end runbook is [../README.md](../README.md).

The VM's Elastic Agent is **not** configured here — the monitoring repo
enrolls it, listing this VM as an agent target.

## Layout

```text
playbooks/
  site.yml               00 then 10
  00-bootstrap.yml       colac.homelab.common + colac.homelab.docker_data
  10-nextcloud.yml       tailscale, nextcloud_aio, reverse_proxy
inventory/               directory inventory
  hosts.yml              hand-authored (git-ignored; see .example)
  group_vars/nextcloud.yml
roles/                   tailscale, nextcloud_aio, reverse_proxy
requirements.yml         colac.homelab pinned to a core tag + Galaxy collections
collections/             installed by `mise run setup` (git-ignored)
```

Run everything from the repo root, through mise:

```bash
mise run inventory nextcloud
mise run ping nextcloud
mise run play nextcloud playbooks/site.yml
mise run play nextcloud playbooks/10-nextcloud.yml --check --diff
```

`mise run play nextcloud` hands `ansible-playbook` the variables from
`nextcloud/secrets.yaml` for that one run — no `-e`, no vault password, and
nothing left in your shell. `group_vars/nextcloud.yml` reads them back with
`lookup('env', …)`. See [Secrets](../../README.md#secrets).

The inventory host is `nextcloud-vm`, not `nextcloud`: Ansible warns and
resolves ambiguously when a host and a group share a name.

## Prerequisites

- The VM exists (`mise run tf nextcloud apply`) and its IP is in
  `inventory/hosts.yml` (`mise run tf nextcloud output ansible_inventory_line`).
- `~/.ssh/homelab-proxmox` can log in as `ubuntu` with passwordless sudo (the
  core template configures this).
- **Tailscale**: MagicDNS enabled; optionally an **auth key** to join
  non-interactively. Route approval and split-DNS come after the first run.
- **Cloudflare**: a token scoped Zone → DNS → Edit on the zone, for DNS-01.
- **PiHole**: a local A record for the Nextcloud hostname → the VM's LAN IP.
- **TrueNAS**: SMB enabled, the share created, a dedicated SMB user (not
  `root`/`admin`) with read/write on it.

## Two passes

Pass 1 deploys AIO + Caddy and joins the tailnet. Then finish AIO's first-run
setup through the SSH tunnel (`ssh -L 8080:localhost:8080 ubuntu@<vm-ip>`,
`https://localhost:8080`). Pass 2 — the same playbook again — creates the
extra accounts and attaches the NAS mounts, which need the Nextcloud container
*running*. Both tasks are idempotent, so re-running is always safe.

## What each role does

- **`tailscale`** — ensures `tailscaled` is running; runs `tailscale up` (if
  an auth key is given) advertising the LAN as a subnet route; keeps the
  route advertised on re-runs.
- **`nextcloud_aio`** — renders `/opt/nextcloud-aio/compose.yaml` and brings up
  the AIO mastercontainer with `docker compose`; the mastercontainer then
  spawns the rest of the AIO stack itself over the Docker socket. Creates the
  accounts in `nextcloud_aio_users` and attaches the NAS shares as external
  storage (below).
- **`reverse_proxy`** — renders and builds a local Caddy image
  (`nextcloud-caddy:local`, Caddy + the `caddy-dns/cloudflare` plugin, both
  pinned in its defaults) that terminates TLS for `nextcloud_domain` via
  Cloudflare DNS-01 and proxies to AIO's Apache container on `127.0.0.1:11000`.
- **`colac.homelab.common` / `colac.homelab.docker_data`** — from the core
  repo; documented there. This VM has no data disk, so `docker_data` skips.

## NAS external storage (SMB)

Nextcloud talks SMB directly to the NAS — no host mount, no extra VM disk. The
`nextcloud_aio` role configures each mount with `occ files_external` inside the
AIO container; each task skips a mount whose name already exists and only runs
once the Nextcloud container is up.

> **Create-only, by design.** The idempotency check keys on the mount *name*
> (`/Familia`), so an existing mount is left completely untouched — a changed
> host, share, `applicable_users`, or `readonly` will **not** be applied to it,
> whether it came from `group_vars` or from a user's `nas_folder`. To change an
> existing mount, delete it and let the next run recreate it (deleting detaches
> only; NAS files are not touched):
>
> ```bash
> docker exec --user www-data nextcloud-aio-nextcloud php occ files_external:list
> docker exec --user www-data nextcloud-aio-nextcloud php occ files_external:delete <id> -y
> ```

`nextcloud_aio_nas_mounts` (in `group_vars/nextcloud.yml`) is a list, one
entry per folder to attach — not one global share. Each item:

```yaml
nextcloud_aio_nas_mounts:
  - name: "Familia"                # folder name shown in Nextcloud Files
    share: "media"                 # SMB share name
    subfolder: "Familia"           # optional: path inside the share
    applicable_users: ["admin"]    # optional: restricts from "all users" to this list
    readonly: true                 # optional: read-only for whoever can see it
```

Mounts are visible to **all** Nextcloud users unless `applicable_users` is
set, and `readonly` applies to everyone who can see the mount — **including
`admin`**, who is not exempt. Both settings are per-*mount*, not per-user, so
granting one account read-write and another read-only on the same NAS path
needs two entries with the same `share`/`subfolder` but different
`name`/`applicable_users`/`readonly`.

The live config deliberately avoids that split: `Familia` is one read-only
mount for everyone. Nextcloud is the viewing and sharing surface for the
curated library; files are moved into it over SMB directly against TrueNAS.
That is not a workaround — a move between two Nextcloud mounts is a
copy-and-delete streamed through the VM (`Common::moveFromStorage` only does a
server-side rename within the *same* storage object), whereas the same move on
TrueNAS is an instant rename inside one dataset. Keeping the mount read-only
therefore costs nothing and removes any chance of a stray drag-and-drop in the
web UI or a desktop sync client deleting curated data.

Credentials (`nextcloud_aio_nas_user`/`nextcloud_aio_nas_password`, and any
per-mount `user`/`password` override) come from the environment —
`nextcloud_nas_user`/`nextcloud_nas_password` in `nextcloud/secrets.yaml` —
never as literals in `group_vars`. To disable the integration entirely, set
`nextcloud_aio_nas_enabled: false`.

## Nextcloud users

`nextcloud_aio_users` creates accounts beyond the AIO admin via idempotent
`occ user:add` calls. It is **not** in `group_vars` as a literal: usernames and
passwords travel together in `nextcloud/secrets.yaml` as
`nextcloud_users_json`, so neither ends up in the tracked repo.

```yaml
# in nextcloud/secrets.yaml (mise run secrets:edit nextcloud) — one line, JSON
nextcloud_users_json: '[{"name":"alice","display_name":"Alice","password":"…","nas_folder":true}]'
```

`display_name` is optional (defaults to `name`); `nas_folder` is optional and
attaches a private NAS folder, see below. JSON rather than nested YAML because
`sops -d --output-type dotenv` emits flat `key=value` pairs only;
`group_vars/nextcloud.yml` restores the structure with `| from_json`. The task
skips any username that already exists, so re-running is safe.

### Private per-user folders

`nas_folder: true` attaches `<nextcloud_aio_nas_personal_share>/<username>` as
a mount named after the user, **scoped to that account only**
(`applicable_users`). The subfolder must already exist on the NAS.

This is why personal folders are not listed in `nextcloud_aio_nas_mounts`:
deriving them from the user list keeps usernames out of the repo, and scoping
is automatic. A personal folder added as a plain mount would default to **all
users** — every account would see and be able to write to it.

## Why Caddy + Cloudflare DNS-01 + a Tailscale subnet route

- AIO's Apache container binds to `127.0.0.1:11000` (`APACHE_IP_BINDING` +
  `APACHE_PORT`); nothing is exposed on the LAN directly by AIO itself.
- Caddy terminates TLS with a real Let's Encrypt cert (DNS-01 via Cloudflare,
  no inbound ports opened) and proxies to that local port.
- `SKIP_DOMAIN_VALIDATION=true` is set because the domain isn't publicly
  reachable on `:443` (private only), so AIO's built-in public validation
  would otherwise fail.
- Tailscale `serve` is deliberately **not** used — it can't present a
  certificate for a custom domain. Instead the VM advertises the LAN as a
  subnet route, and PiHole split-DNS lets tailnet devices resolve the real
  hostname to the LAN IP. Cloudflare Tunnel was rejected too (public by
  default, terminates TLS itself, upload cap).

## Day-2

```bash
ssh ubuntu@<vm-ip> 'cd /opt/nextcloud-aio && sudo docker compose ps'
ssh ubuntu@<vm-ip> 'cd /opt/nextcloud-caddy && sudo docker compose ps'
ssh ubuntu@<vm-ip> 'sudo docker logs nextcloud-caddy'     # cert issues
# AIO updates & backups are managed from the admin UI (port 8080 via tunnel)
```
