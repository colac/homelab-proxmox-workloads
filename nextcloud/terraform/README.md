# Nextcloud VM (Terraform project)

Clones the core repo's `ubuntu-24.04-template` into a VM sized for Nextcloud,
using the core repo's `base-vm` module pinned to a release tag. State is
stored in the Terraform Cloud `Nextcloud` workspace of the `colac_homelab`
organization (Local execution).

```bash
mise run tf nextcloud init
mise run tf nextcloud plan
mise run tf nextcloud apply
mise run tf nextcloud output ansible_inventory_line   # into nextcloud/ansible/inventory/hosts.yml
```

No `terraform.tfvars` is needed: `.mise/sops-exec terraform` supplies
`TF_VAR_pm_api_*` from the repo-root `secrets.yaml` for each command.
`terraform.tfvars.example` documents the non-secret overrides only.

Everything running on the VM is Ansible's job — see
[../ansible/README.md](../ansible/README.md) and the
[Nextcloud runbook](../README.md).

<!-- The generated table shows the git:: module source as a bare URL. -->
<!-- markdownlint-disable MD034 -->
<!-- BEGIN_TF_DOCS -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | ~> 1.15.7 |
| <a name="requirement_proxmox"></a> [proxmox](#requirement\_proxmox) | 3.0.2-rc07 |

## Providers

No providers.

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_nextcloud"></a> [nextcloud](#module\_nextcloud) | git::https://github.com/colac/homelab-proxmox.git//terraform/modules/base-vm | v2.0.0 |

## Resources

No resources.

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_cpu_cores"></a> [cpu\_cores](#input\_cpu\_cores) | Number of vCPUs for the Nextcloud VM. | `number` | `4` | no |
| <a name="input_disk0_size"></a> [disk0\_size](#input\_disk0\_size) | Root disk size (include a unit, e.g. 64G). | `string` | `"64G"` | no |
| <a name="input_memory_mb"></a> [memory\_mb](#input\_memory\_mb) | Memory in MB for the Nextcloud VM. | `number` | `8192` | no |
| <a name="input_network_bridge"></a> [network\_bridge](#input\_network\_bridge) | Proxmox network bridge to attach the VM to. | `string` | `"vmbr0"` | no |
| <a name="input_pm_api_token_id"></a> [pm\_api\_token\_id](#input\_pm\_api\_token\_id) | This is an API token you have previously created for a specific user. | `string` | n/a | yes |
| <a name="input_pm_api_token_secret"></a> [pm\_api\_token\_secret](#input\_pm\_api\_token\_secret) | This uuid is only available when the token was initially created. | `string` | n/a | yes |
| <a name="input_pm_api_url"></a> [pm\_api\_url](#input\_pm\_api\_url) | This is the target Proxmox API endpoint. | `string` | n/a | yes |
| <a name="input_pm_tls_insecure"></a> [pm\_tls\_insecure](#input\_pm\_tls\_insecure) | Skip TLS verification against the Proxmox API. Set via TF\_VAR\_pm\_tls\_insecure by .mise/sops-exec (PROXMOX\_TLS\_INSECURE in mise.toml); true is only needed when the endpoint serves a self-signed certificate. | `bool` | `false` | no |
| <a name="input_proxmox_node"></a> [proxmox\_node](#input\_proxmox\_node) | Proxmox node to deploy the VM on. | `string` | `"pve"` | no |
| <a name="input_proxmox_pool"></a> [proxmox\_pool](#input\_proxmox\_pool) | Optional Proxmox resource pool. | `string` | `null` | no |
| <a name="input_proxmox_storage"></a> [proxmox\_storage](#input\_proxmox\_storage) | Proxmox storage pool for the VM disk and cloud-init drive. | `string` | `"local-lvm"` | no |
| <a name="input_ssh_public_key"></a> [ssh\_public\_key](#input\_ssh\_public\_key) | Path to the SSH public key authorized on the VM. | `string` | `"~/.ssh/homelab-proxmox.pub"` | no |
| <a name="input_template_name"></a> [template\_name](#input\_template\_name) | Name of the Proxmox template to clone. | `string` | `"ubuntu-24.04-template"` | no |
| <a name="input_vm_name"></a> [vm\_name](#input\_vm\_name) | Name of the Nextcloud VM. | `string` | `"nextcloud"` | no |
| <a name="input_vm_user"></a> [vm\_user](#input\_vm\_user) | Cloud-init username created on the VM. | `string` | `"ubuntu"` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_ansible_inventory_line"></a> [ansible\_inventory\_line](#output\_ansible\_inventory\_line) | Ready-to-paste line for the Ansible inventory (see ansible/inventory). |
| <a name="output_vm_ip"></a> [vm\_ip](#output\_vm\_ip) | IP assigned to the deployed Nextcloud VM. |
| <a name="output_vm_name"></a> [vm\_name](#output\_vm\_name) | Name of the deployed Nextcloud VM. |
<!-- END_TF_DOCS -->
