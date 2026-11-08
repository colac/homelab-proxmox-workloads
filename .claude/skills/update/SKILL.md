---
name: update
description: Update an app's VM with this repo's update procedure — OS packages, Nextcloud AIO's own update, reboot — and prepare the pinned Caddy bump. Use when asked to update or reboot Nextcloud or its VM, or when the homelab-update skill gets to workloads.
---

# Update workloads: an app's VM and its apps

The procedure is core's `colac.homelab.update` at each app's pinned tag (core's
`docs/UPDATES.md`). Every command takes the app first; today that is
`nextcloud` (k3s has no VM).

## OS packages, AIO and reboots (no secrets)

Run as background jobs with the output in a scratchpad log, and give the user
the path to `tail -f`. Use `mise -C ~/git-repos/homelab-proxmox-workloads run …`.

| Step | Command | Changes |
|---|---|---|
| Report | `mise run update nextcloud` | Nothing. JSON in `nextcloud/ansible/reports/latest.json` |
| Apply | `mise run update:apply nextcloud` | Snapshot, apt, AIO's update script, health |
| Reboot | add `-e update_reboot=true` | Reboots when the kernel or a library needs it |
| Force AIO | add `-e '{"update_force": ["Nextcloud AIO"]}'` | Runs AIO's update although it reports current |

- **Downtime:** Nextcloud is down for a few minutes during AIO's update and a
  reboot. The VM is also the Tailscale subnet router, so remote access to the
  LAN drops while it restarts.
- **AIO restarts itself.** AIO's mastercontainer updates itself first, which
  stops the first pass: the log shows `ASYNC FAILED` on that pass. The role
  waits for the mastercontainer and runs the update again, so that is
  expected, not a failure.
- **On a failure** the run stops and prints `qm rollback <vmid> <snapshot>`.
  Never roll back yourself: show the user the failing task and the command.

## Caddy: a pinned bump, then the user runs the play

1. Read Caddy's release notes for every version since the pin. Security fixes
   are often marked as possibly breaking, so compare them with our Caddyfile
   (`nextcloud/ansible/roles/reverse_proxy/templates/Caddyfile.j2`: one
   `reverse_proxy` with Cloudflare DNS-01).
2. Check `caddy:<version>-builder` and `caddy:<version>` exist on Docker Hub,
   and that the `caddy-dns/cloudflare` pin is current.
3. On a branch, bump `reverse_proxy_caddy_version` in
   `nextcloud/ansible/roles/reverse_proxy/defaults/main.yml`, then `mise run lint`.
4. The user runs `mise run play nextcloud playbooks/10-nextcloud.yml`. It needs
   credentials, so never run it yourself. Then `mise run update nextcloud`
   should show Caddy `ok`. Commit only when the user asks.
