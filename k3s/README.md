# k3s

A single-node Kubernetes VM — provisioned, not yet configured. The future home
for containerised apps that outgrow one-VM-per-app.

| | |
|---|---|
| Template | `ubuntu-26.04-template` (core repo) |
| Size | 2 vCPU, 8 GB, 24G OS disk + 12G data disk (`docker-vg`) |
| State | TFC workspace `k3s`, Local execution |
| Configuration | none yet — no `ansible/`, no `secrets.yaml` |

```bash
mise run tf k3s plan
mise run tf k3s apply
```

Inputs and outputs: [terraform/README.md](terraform/README.md).

## When it gets configured

Follow "Adding an app" in the [repo README](../README.md#adding-an-app):
an `ansible/` folder starting from `nextcloud/ansible`'s `00-bootstrap.yml`
(which brings in `colac.homelab.docker_data` for the 12G data disk), a
`secrets.yaml` and `sops-exec` profile only if it needs credentials, a
`CLAUDE.md` for its decisions, and an entry in the monitoring repo's agent
targets.
