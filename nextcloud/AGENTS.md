# AGENTS.md — Nextcloud

Agent instructions for `nextcloud/` (Claude Code loads this via `CLAUDE.md`
when working here). The repo-wide rules — secrets, commands, conventions — are
in [../AGENTS.md](../AGENTS.md); the runbook is [README.md](README.md).

Commands take the app first: `mise run play nextcloud playbooks/10-nextcloud.yml
--check --diff`, `mise run tf nextcloud plan` (both need credentials — ask the
human first).

## Current design

AIO in reverse-proxy mode (`APACHE_PORT=11000`, localhost-bound), served
privately at `https://nextcloud.example.com`. TLS is terminated by a **Caddy**
reverse proxy on the VM using a Let's Encrypt cert via **Cloudflare DNS-01**
(no public exposure; Cloudflare only answers DNS). Tailscale provides off-LAN
reach via a `--advertise-routes` subnet route + PiHole split-DNS.

```text
terraform/   VM "nextcloud", ubuntu-24.04-template, 4 vCPU / 8 GB / 64 GB, no
             data disk; TFC workspace "Nextcloud"
ansible/     00-bootstrap (colac.homelab common + docker_data), 10-nextcloud
             (tailscale, nextcloud_aio, reverse_proxy); host `nextcloud-vm`
secrets.yaml nextcloud_domain, acme_email, cloudflare_dns_api_token,
             nextcloud_nas_user/_password, nextcloud_users_json, tailscale_authkey
```

## Decisions that are deliberate (do not "fix" these)

- **App config is Ansible's, not cloud-init's.** Long cloud-init payloads
  through Proxmox hung or timed out on first boot; cloud-init only creates the
  login user and injects the SSH key.
- **`tailscale serve` is not used** — it cannot present a custom-domain cert.
  **Cloudflare Tunnel was rejected** (public by default, MITMs traffic, upload
  cap). The AIO domain submit and the Tailscale route/split-DNS approvals are
  unavoidable one-time manual steps.
- **The AIO domain is permanent.** Whatever is submitted in the AIO UI cannot
  change later; it must match `nextcloud_domain`, the Caddy vhost and the
  PiHole record exactly.
- **AIO stays on `nextcloud/all-in-one:latest`.** AIO supports only `latest`
  (or `beta`) for the mastercontainer, which pins and updates everything it
  manages. Caddy and its `caddy-dns/cloudflare` plugin *are* pinned, in
  `ansible/roles/reverse_proxy/defaults/main.yml`.
- **This app's Cloudflare token is its own.** Kibana's certificate (monitoring
  repo) uses a separate token, so rotating one cannot break the other. Steps:
  core's `docs/CREDENTIALS.md` § Cloudflare DNS tokens. Never paste the token
  into a role default or the rendered Caddy `.env`.
- **Storage lives on TrueNAS as SMB external storage, never on the VM disk.**
  Each person in `nextcloud_users_json` with `nas_folder: true` gets
  `media/<username>` mounted privately (`applicable_users`) — the phone
  auto-upload target. `Familia` is one **read-only** mount for everyone,
  `admin` included: curating it happens over SMB against TrueNAS, where it is
  an instant same-dataset rename instead of the copy-and-delete Nextcloud would
  do across mounts.
- **Mounts and accounts are create-only.** The role never updates or removes
  an existing mount or user, so changing a mount's host, scoping or read-only
  flag means `occ files_external:delete <id> -y` first, then re-run — and a
  rotated SMB or user password in `secrets.yaml` does not reach existing ones
  (see CREDENTIALS.md).
- **Read-only `command` tasks carry `check_mode: false`** so `--check` can
  evaluate the tasks that parse their output. Keep it on new read-only lookups,
  never on a task that changes something.
- **Two passes are normal.** Users and NAS mounts need the Nextcloud container
  running, which only happens after AIO's first-run setup.
- **No data disk.** The VM predates the split OS/data disk design and stays on
  the 24.04 template at 64G; `docker_data` skips it. Moving it to 26.04 is a
  `template_name` change, a re-clone, and an AIO restore — tracked in
  [../TODO.md](../TODO.md), not a drive-by.
- **AIO's Borgbackup does not cover the SMB mounts.** `Familia` exists only on
  TrueNAS; its safety is TrueNAS snapshots plus an off-box copy.
