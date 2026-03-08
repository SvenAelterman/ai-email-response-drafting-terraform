# Contract: Terraform Operator Interface

**Type**: Infrastructure-as-Code operator interface
**Direction**: Operator → Terraform configuration

This document defines the formal interface between an infrastructure operator and the
AI Email Drafting Terraform workload. It specifies required inputs, validation rules,
and the outputs produced after a successful `terraform apply`.

---

## Input Contract (`terraform.tfvars`)

### Required Inputs

| Variable | Type | Constraints | Description |
| --- | --- | --- | --- |
| `location` | string | Must equal `"canadacentral"` | Deployment region |
| `workload` | string | Lowercase letters and hyphens; ≤ 12 chars | Workload name segment (e.g. `"emaildraft"`) |
| `environment` | string | Lowercase letters; ≤ 8 chars | Environment segment (e.g. `"prod"`) |
| `region_abbrev` | string | Lowercase letters; ≤ 6 chars | Region abbreviation (e.g. `"cc"`) |
| `instance` | string | Zero-padded number; e.g. `"001"` | Instance number segment |
| `org` | string | Lowercase alphanumeric; ≤ 10 chars | Organisation identifier segment |
| `vnet_resource_id` | string | Valid Azure resource ID | Resource ID of existing virtual network |
| `snet_aifd_cidr` | string | Valid IPv4 CIDR, available in VNet | Address prefix for AI Foundry PE subnet |
| `snet_srch_cidr` | string | Valid IPv4 CIDR, available in VNet | Address prefix for AI Search PE subnet |
| `snet_st_cidr` | string | Valid IPv4 CIDR, available in VNet | Address prefix for Storage Account PE subnet |
| `snet_kv_cidr` | string | Valid IPv4 CIDR, available in VNet | Address prefix for Key Vault PE subnet |
| `approved_source_cidrs` | list(string) | Non-empty; each entry is valid IPv4 CIDR | NSG allow-inbound source CIDRs |
| `private_dns_rg_name` | string | Resource group must exist | Resource group containing private DNS zones |
| `log_analytics_workspace_resource_id` | string | Valid Azure resource ID; resource must exist | Existing Log Analytics Workspace |
| `llm_model_name` | string | Must be available in `canadacentral` | OpenAI model name (e.g. `"gpt-4o"`) |
| `llm_model_version` | string | Must be a published model version | Model version (e.g. `"2024-05-13"`) |
| `llm_deployment_capacity` | number | Integer > 0 | Capacity in thousands of TPM |

### Optional Inputs

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `tags` | map(string) | `{}` | Resource tags applied to all resources |

### Validation Rules Enforced in `variables.tf`

- `approved_source_cidrs` must have `length(var.approved_source_cidrs) > 0` — if empty, `terraform apply` fails with a validation error
- `llm_deployment_capacity` must be `> 0`

---

## Prerequisite Contract

The following Azure resources **must exist before** `terraform apply` is invoked:

| Resource | How referenced |
| --- | --- |
| Virtual network | `var.vnet_resource_id` |
| Private DNS zone: `privatelink.search.windows.net` | `var.private_dns_rg_name` |
| Private DNS zone: `privatelink.blob.core.windows.net` | `var.private_dns_rg_name` |
| Private DNS zone: `privatelink.vaultcore.azure.net` | `var.private_dns_rg_name` |
| Private DNS zone: `privatelink.cognitiveservices.azure.com` | `var.private_dns_rg_name` |
| Log Analytics Workspace | `var.log_analytics_workspace_resource_id` |
| Terraform remote state backend | Backend config supplied at `terraform init` time |

If any prerequisite is absent, `terraform plan` or `terraform apply` will fail with a
descriptive `data source not found` or `resource not found` error.

---

## Output Contract (`outputs.tf`)

These values are available after a successful `terraform apply` via `terraform output`:

| Output | Description | Sensitive |
| --- | --- | --- |
| `resource_group_name` | Name of the provisioned resource group | No |
| `uami_resource_id` | Resource ID of the user-assigned managed identity | No |
| `uami_principal_id` | Principal (object) ID of the UAMI — needed for RBAC verification | No |
| `key_vault_resource_id` | Resource ID of the Key Vault | No |
| `storage_account_resource_id` | Resource ID of the Storage Account | No |
| `ai_search_resource_id` | Resource ID of the AI Search service | No |
| `foundry_hub_resource_id` | Resource ID of the Microsoft AI Foundry Hub account | No |
| `foundry_project_resource_id` | Resource ID of the Microsoft AI Foundry Project | No |

> **Security invariant**: No output may expose a storage account key, a SAS token, an AI
> Search admin key, an AI Foundry API key, or any other secret credential. All cross-service
> authentication uses managed identity (UAMI). Any output that would expose such a value is
> prohibited by FR-016 and SC-007.

---

## Operator Workflow Contract

The operator MUST follow this sequence on every change to the configuration:

1. `terraform fmt -check` → must exit 0 (FR-020)
2. `terraform init` (if first run or provider/module versions changed)
3. `terraform validate` → must exit 0, zero errors (FR-020, SC-001)
4. `terraform plan -out=tfplan` → review plan for unexpected changes (SC-002)
5. `terraform apply tfplan` → deploy (SC-003)

Skipping steps 1–4 is a constitution violation and is prohibited.
