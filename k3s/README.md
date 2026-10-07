# k3s

A single-node Kubernetes VM — **not deployed**. The VM was deleted from
Proxmox; this folder is the recipe to create it again, and the future home for
containerised apps that outgrow one-VM-per-app.

| | |
|---|---|
| Template | `ubuntu-26.04-template` (core repo) |
| Size | 2 vCPU, 8 GB, 24G OS disk + 12G data disk (`docker-vg`) |
| State | TFC workspace `k3s`, Local execution |
| Configuration | none yet — no `ansible/`, no `secrets.yaml` |

The Terraform Cloud state may still list the deleted VM, so a plan proposes
creating it. Either let the state catch up with reality, or create the VM:

```bash
mise run tf k3s init
mise run tf k3s apply -refresh-only   # forget the deleted VM; nothing is created
mise run tf k3s apply                 # or: create a fresh VM from the recipe
```

Inputs and outputs: [terraform/README.md](terraform/README.md).

## When it gets configured

Follow "Adding an app" in the [repo README](../README.md#adding-an-app):
an `ansible/` folder starting from `nextcloud/ansible`'s `00-bootstrap.yml`
(which brings in `colac.homelab.docker_data` for the 12G data disk), a
`secrets.yaml` and `sops-exec` profile only if it needs credentials, a
`AGENTS.md` for its decisions, and an entry in the monitoring repo's agent
targets.
