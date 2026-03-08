# Quickstart: 001-ai-email-drafting-workload

This guide walks an infrastructure engineer through deploying the AI Email Drafting workload
from scratch using Terraform, AVM modules, and a configured `terraform.tfvars`.

---

## Prerequisites

| Requirement | Notes |
| --- | --- |
| Terraform `~> 1.11` | Download from <https://developer.hashicorp.com/terraform/install> |
| Azure CLI `>= 2.60` | `az login` with a subscription containing Contributor + User Access Administrator |
| Existing virtual network | Record its resource ID |
| Existing private DNS zones (6 zones) | Must be in a single resource group; record the RG name |
| Existing Log Analytics Workspace | Record its resource ID |
| Azure OpenAI quota for `gpt-4o` in `canadacentral` | Submit a quota request in Azure portal if not yet approved |
| Terraform remote state backend | Configure in `backend.tf` or via environment variables before `terraform init` |

---

## Step 1 — Clone and inspect the repository

```bash
git clone <repo-url>
cd ai-email-response-drafting-terraform
git checkout 001-ai-email-drafting-workload
```

The `src/` directory will contain the root Terraform module:

```plain
src/
├── main.tf           # All module calls and azapi resources
├── variables.tf      # All variable declarations with descriptions and validation
├── outputs.tf        # Post-deploy output values (resource IDs only, no secrets)
├── terraform.tf      # required_version and required_providers pinning
└── terraform.tfvars  # Operator-supplied values (NEVER commit secrets here)
```

---

## Step 2 — Configure `terraform.tfvars`

Copy and adapt the example below. All values must match your environment.

```hcl
# --- Region ---
location     = "canadacentral"
region_abbrev = "cc"

# --- Six-segment naming ---
# Pattern: <workload>-<environment>-<type>-<region>-<instance>-<org>
workload     = "emaildraft"
environment  = "prod"
instance     = "001"
org          = "contoso"

# --- Existing infrastructure ---
vnet_resource_id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-hub-network/providers/Microsoft.Network/virtualNetworks/vnet-hub-cc"

private_dns_rg_name = "rg-privatedns-shared"

log_analytics_workspace_resource_id = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-central"

# --- PE subnet CIDRs (must be unused ranges within the existing VNet) ---
snet_aifd_cidr = "10.0.16.0/27"   # AI Foundry Hub private endpoint subnet
snet_srch_cidr = "10.0.16.64/27"  # AI Search private endpoint subnet
snet_st_cidr   = "10.0.16.96/27"  # Storage Account private endpoint subnet
snet_kv_cidr   = "10.0.16.128/27" # Key Vault private endpoint subnet

# --- NSG inbound allow rules ---
# At least one CIDR must be provided; apply fails with a validation error if empty.
approved_source_cidrs = [
  "10.0.0.0/8",    # Internal corporate network
]

# --- OpenAI model deployment ---
llm_model_name          = "gpt-4o"
llm_model_version       = "2024-05-13"
llm_deployment_capacity = 8  # 8,000 tokens per minute (TPM)

# --- Optional: resource tags ---
tags = {
  workload    = "email-drafting"
  environment = "prod"
  cost-centre = "ai-innovation"
}
```

> **Security note**: `terraform.tfvars` must never contain API keys, passwords, or connection
> strings. All cross-service authentication uses managed identity. Do not commit this file to
> version control if it contains sensitive subnet CIDR ranges or internal resource IDs.

---

## Step 3 — Initialise Terraform

```bash
cd src/
terraform init
```

This downloads all AVM modules and providers pinned in `terraform.tf`. Ensure your backend
is configured before running `init` (remote state is handled outside this workload).

---

## Step 4 — Validate before deploy (FR-020 / constitution requirement)

```bash
# Check Terraform formatting
terraform fmt -check

# Validate configuration
terraform validate
```

Both commands must exit with code **0** and zero errors before continuing.

---

## Step 5 — Review the plan

```bash
terraform plan -out=tfplan
```

Review the output and verify:

- Only resources in the expected set appear (no unexpected additions)
- Resource names follow the six-segment convention
- No secrets appear in plan output

Expected resource count: **≈30–35 resources** (modules + azapi resources + private endpoints

- DNS zone groups + role assignments)

---

## Step 6 — Apply

```bash
terraform apply tfplan
```

Apply takes approximately **15–20 minutes** because:

- AI Foundry Hub managed-VNet provisioning (`provision_network_now_enabled = true`) can take
  10+ minutes
- Private endpoints and DNS group registration require sequential provisioning

---

## Step 7 — Verify the deployment

```bash
# List all deployed resource IDs
terraform output
```

Then confirm in the Azure portal:

| Check | Expected result |
| --- | --- |
| Resource group exists in `canadacentral` | Named `emaildraft-prod-rg-cc-001-contoso` (or your values) |
| Microsoft AI Foundry Hub | CognitiveServices account kind=AIServices; public network access disabled; PE visible |
| AI Search | S1 tier, semantic ranking enabled, public access off, PE visible |
| Storage Account | ZRS replication, public access off, PE visible |
| Key Vault | Standard SKU, public access off, PE visible |
| GPT-4o deployment | Visible under the Foundry Hub account → Model deployments |
| AI Search indexer | Status: Running or Idle; next run within 1 hour |
| Diagnostic settings | Visible on all services, pointing to the Log Analytics Workspace |
| RBAC | UAMI holds 3 role assignments (Search ×2, Storage ×1) |

---

## Tear Down

```bash
terraform destroy
```

> **Warning**: `terraform destroy` permanently deletes all provisioned resources including
> the resource group, all AI services, all data in the Storage Account, all Key Vault secrets,
> and the AI Search index. This operation cannot be undone. Ensure all data has been backed up
> before proceeding.

---

## Troubleshooting

| Symptom | Likely cause | Resolution |
| --- | --- | --- |
| `terraform validate` fails with "namespace not found" | AVM module not found in registry | Run `terraform init -upgrade` to refresh module cache |
| `Error: data source not found` for a private DNS zone | Zone missing from `var.private_dns_rg_name` | Create the missing zone and link it to the VNet |
| AI Foundry Hub apply times out (30 min) | CognitiveServices resource provisioning in `canadacentral` is slow | Retry `terraform apply`; the resource will continue provisioning |
| AI Search indexer shows `Transient Error` | Storage PE not yet resolvable from indexer | Wait 5 minutes and re-trigger the indexer via Azure portal |
| `terraform plan` shows unexpected role assignments | Running user has User Access Administrator | Expected; role assignments appear in plan on first run |
