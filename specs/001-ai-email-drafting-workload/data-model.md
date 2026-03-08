# Data Model: 001-ai-email-drafting-workload

**Phase**: 1 — Design
**Source**: Derived from `spec.md` + research findings in `research.md`

This document enumerates every Terraform entity in the AI Email Drafting workload: data
sources, locals, module calls, azapi resources, random resources, and outputs. It also defines
the variable schema, dependency chain, and RBAC assignment matrix.

---

## 1. Data Sources

| Terraform Identifier | Resource Type | Purpose |
| --- | --- | --- |
| `data.azurerm_virtual_network.existing` | `azurerm_virtual_network` | Existing VNet referenced by `var.vnet_resource_id` |
| `data.azurerm_private_dns_zone.search` | `azurerm_private_dns_zone` | `privatelink.search.windows.net` |
| `data.azurerm_private_dns_zone.blob` | `azurerm_private_dns_zone` | `privatelink.blob.core.windows.net` |
| `data.azurerm_private_dns_zone.keyvault` | `azurerm_private_dns_zone` | `privatelink.vaultcore.azure.net` |
| `data.azurerm_private_dns_zone.cognitiveservices` | `azurerm_private_dns_zone` | `privatelink.cognitiveservices.azure.com` |

All DNS zone data sources use `resource_group_name = var.private_dns_rg_name`.

---

## 2. Local Values (Naming and Shared Config)

| Local | Expression | Purpose |
| --- | --- | --- |
| `local.rg_name` | `"${var.workload}-${var.environment}-rg-${var.region_abbrev}-${var.instance}-${var.org}"` | Resource group |
| `local.uami_name` | `"${var.workload}-${var.environment}-id-${var.region_abbrev}-${var.instance}-${var.org}"` | User-assigned managed identity |
| `local.kv_name` | `"${var.workload}-${var.environment}-kv-${var.region_abbrev}-${var.instance}-${var.org}"` | Key Vault |
| `local.st_base_name` | Abbreviated segments, ≤ 20 chars | Storage Account base (before suffix) |
| `local.st_name` | `"${replace(local.st_base_name, "-", "")}${random_string.storage_suffix.result}"` | Storage Account (max 24 chars, no hyphens) |
| `local.search_name` | `"${var.workload}-${var.environment}-srch-${var.region_abbrev}-${var.instance}-${var.org}"` | AI Search |
| `local.foundry_name` | `"${var.workload}-${var.environment}-aifd-${var.region_abbrev}-${var.instance}-${var.org}"` | Microsoft AI Foundry Hub (also used as custom subdomain) |
| `local.project_name` | `"${var.workload}-${var.environment}-aiproj-${var.region_abbrev}-${var.instance}-${var.org}"` | AI Foundry Project |
| `local.nsg_aifd_name` | `"${var.workload}-${var.environment}-nsg-aifd-${var.region_abbrev}-${var.instance}-${var.org}"` | NSG for AI Foundry PE subnet |
| `local.nsg_srch_name` | `"${var.workload}-${var.environment}-nsg-srch-${var.region_abbrev}-${var.instance}-${var.org}"` | NSG for AI Search PE subnet |
| `local.nsg_st_name` | `"${var.workload}-${var.environment}-nsg-st-${var.region_abbrev}-${var.instance}-${var.org}"` | NSG for Storage PE subnet |
| `local.nsg_kv_name` | `"${var.workload}-${var.environment}-nsg-kv-${var.region_abbrev}-${var.instance}-${var.org}"` | NSG for Key Vault PE subnet |
| `local.snet_aifd_name` | `"${var.workload}-${var.environment}-snet-aifd-${var.region_abbrev}-${var.instance}-${var.org}"` | AI Foundry PE subnet |
| `local.snet_srch_name` | `"${var.workload}-${var.environment}-snet-srch-${var.region_abbrev}-${var.instance}-${var.org}"` | AI Search PE subnet |
| `local.snet_st_name` | `"${var.workload}-${var.environment}-snet-st-${var.region_abbrev}-${var.instance}-${var.org}"` | Storage PE subnet |
| `local.snet_kv_name` | `"${var.workload}-${var.environment}-snet-kv-${var.region_abbrev}-${var.instance}-${var.org}"` | Key Vault PE subnet |
| `local.diag_settings` | Common diagnostic_settings map | Re-used in every module call |

---

## 3. Random Resources

| Identifier | Type | Spec | Purpose |
| --- | --- | --- | --- |
| `random_string.storage_suffix` | `random_string` | length=4, lower=true, upper=false, special=false | Global uniqueness suffix for Storage Account |

---

## 4. AVM Module Calls

### 4.1 Resource Group

| Module key | `module.resource_group` |
| --- | --- |
| Source | `Azure/avm-res-resources-resourcegroup/azurerm` |
| Version | `0.2.2` |
| Key inputs | `name = local.rg_name`, `location = var.location`, `tags = var.tags` |

### 4.2 User-Assigned Managed Identity

| Module key | `module.uami` |
| --- | --- |
| Source | `Azure/avm-res-managedidentity-userassignedidentity/azurerm` |
| Version | `0.4.0` |
| Key inputs | `name = local.uami_name`, `resource_group_name = module.resource_group.name`, `location`, `tags` |

### 4.3 Key Vault

| Module key | `module.key_vault` |
| --- | --- |
| Source | `Azure/avm-res-keyvault-vault/azurerm` |
| Version | `0.10.2` |
| Key inputs | `name = local.kv_name`, `sku_name = "standard"`, `public_network_access_enabled = false`, `diagnostic_settings = local.diag_settings` |
| Private endpoints | `kv_pe` → `snet_kv` subnet, `privatelink.vaultcore.azure.net` |
| Role assignments | None beyond AVM defaults |

### 4.4 Storage Account

| Module key | `module.storage_account` |
| --- | --- |
| Source | `Azure/avm-res-storage-storageaccount/azurerm` |
| Version | `0.6.7` |
| Key inputs | `name = local.st_name`, `account_replication_type = "ZRS"`, `shared_access_key_enabled = false`, `public_network_access_enabled = false`, `diagnostic_settings = local.diag_settings` |
| Private endpoints | `blob_pe` → `snet_st` subnet, `privatelink.blob.core.windows.net` |
| Role assignments | `Storage Blob Data Contributor` → `module.uami.principal_id` |

### 4.5 AI Search

| Module key | `module.ai_search` |
| --- | --- |
| Source | `Azure/avm-res-search-searchservice/azurerm` |
| Version | `0.2.0` |
| Key inputs | `name = local.search_name`, `sku = "standard"` (S1), `semantic_search_sku = "standard"`, `replica_count = 1`, `partition_count = 1`, `public_network_access_enabled = false`, `local_authentication_enabled = false`, `diagnostic_settings = local.diag_settings` |
| Private endpoints | `search_pe` → `snet_srch` subnet, `privatelink.search.windows.net` |
| Role assignments | `Search Index Data Contributor` + `Search Service Contributor` → `module.uami.principal_id` |

### 4.6 Microsoft AI Foundry Hub

> **Provisioned via `azapi_resource`** — no AVM module available. See Section 5 for the full
> azapi block definition.

| Field | Value |
| --- | --- |
| Identifier | `azapi_resource.foundry_hub` |
| Resource type | `Microsoft.CognitiveServices/accounts@2025-04-01-preview` |
| Key properties | `kind = "AIServices"`, `sku.name = "S0"`, `publicNetworkAccess = "Disabled"`, `networkAcls.defaultAction = "Deny"`, `customSubDomainName = local.foundry_name` |
| Identity | `type = "UserAssigned"`, `userAssignedIdentities = { "<uami_id>" = {} }` |
| Private endpoint | `azapi_resource.pe_foundry` → `snet_aifd` subnet, sub-resource `"account"`, DNS zone `privatelink.cognitiveservices.azure.com` |
| Diagnostics | `azapi_resource.foundry_hub_diag` (separate `Microsoft.Insights/diagnosticSettings`) |

### 4.7 Microsoft AI Foundry Project

> **Provisioned via `azapi_resource`** — sub-resource of the Foundry Hub. See Section 5.

| Field | Value |
| --- | --- |
| Identifier | `azapi_resource.foundry_project` |
| Resource type | `Microsoft.CognitiveServices/accounts/projects@2025-04-01-preview` |
| Parent | `azapi_resource.foundry_hub.id` |
| Key properties | `sku.name = "S0"` |
| Identity | `type = "UserAssigned"`, `userAssignedIdentities = { "<uami_id>" = {} }` |
| Note | Project is a sub-resource; no separate PE — accessible through Hub account PE |

### 4.8 Network Security Groups (×4)

Each NSG module follows the same pattern; one per PE subnet:

| Module key | Subnet protected |
| --- | --- |
| `module.nsg_aifd` | AI Foundry Hub PE subnet |
| `module.nsg_srch` | AI Search PE subnet |
| `module.nsg_st` | Storage Account PE subnet |
| `module.nsg_kv` | Key Vault PE subnet |

**Source**: `Azure/avm-res-network-networksecuritygroup/azurerm` v0.5.1  
**Common inputs**:

- `resource_group_name = module.resource_group.name`
- `location = var.location`
- `tags = var.tags`
- `security_rules` map with one allow-inbound rule per CIDR in `var.approved_source_cidrs`
  and a default-deny-all-inbound rule

---

## 5. azapi Resources

| Identifier | AzAPI Type & API Version | Parent | Purpose |
| --- | --- | --- | --- |
| `azapi_resource.snet_aifd` | `Microsoft.Network/virtualNetworks/subnets@2024-01-01` | existing VNet | AI Foundry Hub PE subnet |
| `azapi_resource.snet_srch` | `Microsoft.Network/virtualNetworks/subnets@2024-01-01` | existing VNet | AI Search PE subnet |
| `azapi_resource.snet_st` | `Microsoft.Network/virtualNetworks/subnets@2024-01-01` | existing VNet | Storage Account PE subnet |
| `azapi_resource.snet_kv` | `Microsoft.Network/virtualNetworks/subnets@2024-01-01` | existing VNet | Key Vault PE subnet |
| `azapi_resource.foundry_hub` | `Microsoft.CognitiveServices/accounts@2025-04-01-preview` | resource group | Microsoft AI Foundry Hub (kind=AIServices) |
| `azapi_resource.foundry_project` | `Microsoft.CognitiveServices/accounts/projects@2025-04-01-preview` | `azapi_resource.foundry_hub` | Microsoft AI Foundry Project |
| `azapi_resource.pe_foundry` | `Microsoft.Network/privateEndpoints@2024-01-01` | resource group | PE for Foundry Hub account (sub-resource `"account"`) |
| `azapi_resource.pe_foundry_dns_group` | `Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2024-01-01` | `azapi_resource.pe_foundry` | Register Foundry Hub PE in `privatelink.cognitiveservices.azure.com` |
| `azapi_resource.foundry_hub_diag` | `Microsoft.Insights/diagnosticSettings@2021-05-01-preview` | `azapi_resource.foundry_hub` | Diagnostics for Foundry Hub → LAW |
| `azapi_resource.gpt4o_deployment` | `Microsoft.CognitiveServices/accounts/deployments@2024-10-01` | `azapi_resource.foundry_hub` | GPT-4o model deployment |
| `azapi_resource.search_index` | `Microsoft.Search/searchServices/indexes@2024-07-01` | `module.ai_search` | Vector-enabled knowledge-base index |
| `azapi_resource.search_datasource` | `Microsoft.Search/searchServices/dataSources@2024-07-01` | `module.ai_search` | Blob storage data source (UAMI auth via `identity.userAssignedIdentity.clientId`) |
| `azapi_resource.search_indexer` | `Microsoft.Search/searchServices/indexers@2024-07-01` | `module.ai_search` | Hourly scheduled indexer; change-detection policy enabled |

### Subnet body fields (all 5 subnets)

```jsonc
{
  "addressPrefix": "<var.snet_*_cidr>",
  "privateEndpointNetworkPolicies": "Disabled",
  "networkSecurityGroup": { "id": "<module.nsg_*.resource_id>" }
}
```

### GPT-4o deployment body

```jsonc
{
  "sku": { "name": "Standard", "capacity": "<var.llm_deployment_capacity>" },
  "model": {
    "format": "OpenAI",
    "name": "<var.llm_model_name>",
    "version": "<var.llm_model_version>"
  }
}
```

### AI Search indexer schedule

```jsonc
{ "schedule": { "interval": "PT1H", "startTime": "2000-01-01T00:00:00Z" } }
```

---

## 6. Variable Schema (`variables.tf`)

| Variable | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `location` | `string` | Yes | — | Azure region (`"canadacentral"`) |
| `workload` | `string` | Yes | — | Workload segment, e.g. `"emaildraft"` |
| `environment` | `string` | Yes | — | Environment segment, e.g. `"prod"` |
| `region_abbrev` | `string` | Yes | — | Region abbreviation, e.g. `"cc"` |
| `instance` | `string` | Yes | — | Instance number, e.g. `"001"` |
| `org` | `string` | Yes | — | Org identifier, e.g. `"contoso"` |
| `vnet_resource_id` | `string` | Yes | — | Resource ID of existing VNet |
| `snet_aifd_cidr` | `string` | Yes | valid CIDR | Address prefix for AI Foundry Hub PE subnet |
| `snet_srch_cidr` | `string` | Yes | valid CIDR | Address prefix for AI Search PE subnet |
| `snet_st_cidr` | `string` | Yes | valid CIDR | Address prefix for Storage Account PE subnet |
| `snet_kv_cidr` | `string` | Yes | valid CIDR | Address prefix for Key Vault PE subnet |
| `approved_source_cidrs` | `list(string)` | Yes | `length > 0` | NSG allow-inbound source CIDRs; must be non-empty |
| `private_dns_rg_name` | `string` | Yes | — | Resource group containing existing private DNS zones |
| `log_analytics_workspace_resource_id` | `string` | Yes | — | Existing Log Analytics Workspace resource ID |
| `llm_model_name` | `string` | Yes | — | OpenAI model name, e.g. `"gpt-4o"` |
| `llm_model_version` | `string` | Yes | — | Model version, e.g. `"2024-05-13"` |
| `llm_deployment_capacity` | `number` | Yes | `> 0` | TPM capacity units for the model deployment |
| `tags` | `map(string)` | No | — | Resource tags; defaults to `{}` |

---

## 7. Resource Dependency Chain

```plain
random_string.storage_suffix
  └── module.resource_group  (location, tags)
        ├── module.uami
        ├── module.nsg_aifd, module.nsg_srch, module.nsg_st, module.nsg_kv
        │     └── (each NSG is associated to the matching subnet via azapi_resource body)
        ├── azapi_resource.snet_aifd, snet_srch, snet_st, snet_kv
        │     (references data.azurerm_virtual_network.existing.id and module.nsg_*.resource_id)
        ├── module.key_vault
        │     └── private_endpoint → azapi_resource.snet_kv
        │     └── private_dns_zone_resource_ids → data.azurerm_private_dns_zone.keyvault
        ├── module.storage_account
        │     ├── role_assignments → module.uami.principal_id
        │     └── private_endpoint → azapi_resource.snet_st → data.azurerm_private_dns_zone.blob
        ├── module.ai_search
        │     ├── role_assignments → module.uami.principal_id
        │     ├── private_endpoint → azapi_resource.snet_srch → data.azurerm_private_dns_zone.search
        │     ├── azapi_resource.search_index
        │     ├── azapi_resource.search_datasource  (refs storage_account, uami client_id)
        │     └── azapi_resource.search_indexer     (refs search_index + search_datasource)
        └── azapi_resource.foundry_hub              (refs resource_group, uami, storage, key_vault)
              ├── identity → module.uami.resource_id
              ├── azapi_resource.pe_foundry → azapi_resource.snet_aifd
              │     └── azapi_resource.pe_foundry_dns_group
              │           → data.azurerm_private_dns_zone.cognitiveservices
              ├── azapi_resource.foundry_hub_diag (diagnostics → LAW)
              ├── azapi_resource.gpt4o_deployment
              └── azapi_resource.foundry_project
                    └── identity → module.uami.resource_id
```

---

## 8. RBAC Assignment Matrix

| AVM Module | `role_assignments` key | Role | Principal |
| --- | --- | --- | --- |
| `module.storage_account` | `"blob_contributor"` | `Storage Blob Data Contributor` | `module.uami.principal_id` |
| `module.ai_search` | `"search_index_contributor"` | `Search Index Data Contributor` | `module.uami.principal_id` |
| `module.ai_search` | `"search_service_contributor"` | `Search Service Contributor` | `module.uami.principal_id` |

> The previous `Cognitive Services OpenAI User` role on a separate Azure OpenAI resource is
> **eliminated** — there is no separate Azure OpenAI resource. The UAMI is the identity of the
> Foundry Hub account itself, so no cross-resource role is required for model inferencing.

---

## 9. Diagnostic Settings (`local.diag_settings`)

All services receive the same AVM diagnostic settings structure:

```hcl
locals {
  diag_settings = {
    main = {
      name                           = "diag-to-law"
      workspace_resource_id          = var.log_analytics_workspace_resource_id
      log_groups                     = ["allLogs"]
      metric_categories              = ["AllMetrics"]
      log_analytics_destination_type = "Dedicated"
    }
  }
}
```

Applied to: `module.key_vault`, `module.storage_account`, `module.ai_search`.
The Microsoft AI Foundry Hub uses a separate `azapi_resource.foundry_hub_diag` targeting
`Microsoft.Insights/diagnosticSettings@2021-05-01-preview` on the Foundry Hub resource ID,
with the same workspace_id and log/metric categories.

---

## 10. Outputs (`outputs.tf`)

| Output name | Value | Sensitive |
| --- | ---  --- |
| `resource_group_name` | `module.resource_group.name` | No |
| `uami_resource_id` | `module.uami.resource_id` | No |
| `uami_principal_id` | `module.uami.principal_id` | No |
| `key_vault_resource_id` | `module.key_vault.resource_id` | No |
| `storage_account_resource_id` | `module.storage_account.resource_id` | No |
| `ai_search_resource_id` | `module.ai_search.resource_id` | No |
| `foundry_hub_resource_id` | `azapi_resource.foundry_hub.id` | No |
| `foundry_project_resource_id` | `azapi_resource.foundry_project.id` | No |

> **No secrets in outputs** — FR-016 and SC-007 prohibit emitting storage keys, API keys, or
> connection strings. No `sensitive = true` outputs are defined; all outputs are resource IDs
> or non-secret identifiers.
