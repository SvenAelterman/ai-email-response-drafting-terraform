# Research: 001-ai-email-drafting-workload

**Date**: 2026-03-08
**Phase**: 0 — Research
**Process**: Resolved all NEEDS CLARIFICATION items from Technical Context; all decisions are final.

---

## 1. Module Strategy

### Decision

Use **individual AVM resource modules** (`avm-res-*`) for all Azure resources that have AVM
coverage. Use **`azapi_resource`** for Microsoft Foundry, subnets, model
deployments, AI Search data-plane resources, and diagnostic settings on azapi-managed resources.
The pattern module `avm-ptn-aiml-ai-foundry` is **not used**.

### Rationale

The workload requires a **Microsoft AI Foundry** instance (not an Azure Machine Learning Hub).
Microsoft AI Foundry uses the CognitiveServices API
(`Microsoft.CognitiveServices/accounts` kind=`AIServices`) and has no AVM resource module at
this time. `azapi_resource` is the constitution-compliant fallback for Azure resources without
AVM coverage. The pattern module `avm-ptn-aiml-ai-foundry` v0.10.1 wraps this API but adds
unwanted resource creation (Key Vault, Storage, and CosmosDB) that should be independently
controlled; using individual modules gives the operator full control over each resource.

### Alternatives Considered

| Alternative | Rejected Because |
| --- | --- |
| `avm-ptn-aiml-ai-foundry` v0.10.1 | Creates Key Vault, Storage, CosmosDB, and UAMI opaquely; operator loses direct control of each resource |
| `avm-res-machinelearningservices-workspace` kind=Hub (ML Hub) | This is an Azure Machine Learning Hub, not Microsoft AI Foundry; different resource type, different surface area |
| `avm-res-cognitiveservices-account` kind=AIServices | Would technically work but user requirement specifies `azapi_resource` since no dedicated Foundry AVM module exists |

---

## 2. Microsoft AI Foundry Hub — azapi_resource

### Decision

- **Mechanism**: `azapi_resource` (no AVM module for Microsoft AI Foundry)
- **Resource type**: `Microsoft.CognitiveServices/accounts@2025-04-01-preview`
- **Kind**: `"AIServices"`
- **SKU**: `"S0"`
- **Network isolation**: `publicNetworkAccess = "Disabled"` + `networkAcls.defaultAction = "Deny"`;
  isolation is enforced via customer-VNet private endpoint (PE sub-resource `"account"`);
  this replaces the ML-specific `AllowOnlyApprovedOutbound` managed VNet (which is not
  available for CognitiveServices resources)
- **Custom subdomain**: required so the PE DNS record resolves to the right endpoint
- **Identity**: user-assigned (`type = "UserAssigned"`, keyed by `module.uami.resource_id`)
- **Customer-VNet PE**: sub-resource `"account"`, DNS zone `privatelink.cognitiveservices.azure.com`
- **Diagnostic settings**: separate `azapi_resource` targeting
  `Microsoft.Insights/diagnosticSettings` on the Foundry Hub (AVM `diagnostic_settings` input
  is not available on azapi-managed resources)

### Rationale

The CognitiveServices API does not support `AllowOnlyApprovedOutbound` managed VNet (that is an
ML workspace concept). Equivalent security is achieved by:

- `publicNetworkAccess = "Disabled"` — no public API surface
- `networkAcls.defaultAction = "Deny"` — control-plane network ACL
- Customer-VNet private endpoint with DNS registration — data-plane is only reachable from inside the VNet

This satisfies the **intent** of FR-002a (no public access, traffic stays within approved
network boundaries) using the mechanisms available on CognitiveServices resources.

---

## 3. Microsoft AI Foundry Project — azapi_resource

### Decision

- **Mechanism**: `azapi_resource` (sub-resource of the Foundry Hub)
- **Resource type**: `Microsoft.CognitiveServices/accounts/projects@2025-04-01-preview`
- **Parent**: `azapi_resource.foundry_hub.id`
- **SKU**: `"S0"`
- **Identity**: user-assigned (`type = "UserAssigned"`, keyed by `module.uami.resource_id`)
- **No separate customer-VNet PE** — the Project is a sub-resource of the Hub account and is
  accessible via the Hub account's PE

### Rationale

Foundry Projects are sub-resources of a Foundry Hub account. The project's API surface is
accessible through the parent Hub's private endpoint. Creating a separate PE for the Project
would be redundant and is not supported.

---

## 4. LLM Deployment on Microsoft AI Foundry Hub (FR-006)

### Decision

- **No separate Azure OpenAI resource** — the Foundry Hub (`kind = "AIServices"`) natively
  supports OpenAI model deployments; a separate CognitiveServices `kind = "OpenAI"` account
  is not needed
- **GPT-4o deployment**: `azapi_resource` targeting
  `Microsoft.CognitiveServices/accounts/deployments@2024-10-01`, parent =
  `azapi_resource.foundry_hub.id`
- **Module `avm-res-cognitiveservices-account` is removed** from the module list

### Rationale

`Microsoft.CognitiveServices/accounts` with `kind = "AIServices"` is the Microsoft AI Foundry
resource type and supports OpenAI model deployments (`gpt-4o`, `gpt-4o-mini`, embeddings, etc.)
directly. Deploying a separate Azure OpenAI account would add an unnecessary resource, a
separate private endpoint and subnet, and a cross-resource RBAC role assignment — all of which
are eliminated by deploying the model on the Foundry Hub account itself.

GPT-4o deployments use `azapi_resource` (no AVM model deployment module exists) — same
reasoning as before, constitution-compliant.

---

## 5. Subnet Strategy

### Decision

Provision **4 dedicated PE subnets** using `azapi_resource`
(`Microsoft.Network/virtualNetworks/subnets@2024-01-01`), one per service group, inside the
**existing VNet** (referenced via `data "azurerm_virtual_network" "existing"`):

| # | Subnet | Service(s) | CIDR Variable |
| --- | --- | --- | --- |
| 1 | `snet-*-aifd-*` | Microsoft AI Foundry Hub PE | `var.snet_aifd_cidr` |
| 2 | `snet-*-srch-*` | AI Search PE | `var.snet_srch_cidr` |
| 3 | `snet-*-st-*` | Storage Account PE | `var.snet_st_cidr` |
| 4 | `snet-*-kv-*` | Key Vault PE | `var.snet_kv_cidr` |

Each subnet has:

- `privateEndpointNetworkPolicies = "Disabled"` (required for PE attachment)
- NSG association: `networkSecurityGroup.id = module.nsg_<service>.resource_id`

Matching NSGs (`avm-res-network-networksecuritygroup` v0.5.1) allow inbound only from
`var.approved_source_cidrs`; all other inbound denied.

### Rationale

`avm-res-network-subnet` does not exist (confirmed: 404 on registry lookup). `azapi_resource`
is the only constitution-compliant option. Separate subnets per service align with FR-005
("each subnet has a dedicated NSG"). The previous `snet_oai` subnet (Azure OpenAI PE) is
eliminated because the OpenAI model deployment now lives on the Foundry Hub account; the
Foundry Hub account PE (`snet_aifd`) covers both the Foundry APIs and the OpenAI endpoint.

---

## 6. AI Search Index, Data Source, and Indexer

### Decision

All AI Search data-plane resources use `azapi_resource`:

| Resource | AzAPI type |
| --- | --- |
| Vector index | `Microsoft.Search/searchServices/indexes@2024-07-01` |
| Blob data source | `Microsoft.Search/searchServices/dataSources@2024-07-01` |
| Scheduled indexer | `Microsoft.Search/searchServices/indexers@2024-07-01` |

The data source authenticates to Storage using the UAMI client ID (managed identity auth,
no connection string) — satisfying FR-016/FR-003a requirements.

### Rationale

AI Search data-plane resources (indexes, data sources, indexers) are management-plane REST
resources not supported by AVM or by the azurerm provider. `azapi_resource` is the only
compliant method. The hourly schedule is expressed as an [`ISO 8601` duration string
`"PT1H"`](https://learn.microsoft.com/azure/search/search-howto-schedule-indexers).

---

## 7. Provider and Module Versions

| Component | Source | Pinned Version |
| --- | --- | --- |
| Terraform | built-in | `~> 1.11` |
| `hashicorp/azurerm` | hashicorp/azurerm | `~> 4.63` |
| `azure/azapi` | azure/azapi | `~> 2.8` |
| `hashicorp/random` | hashicorp/random | `~> 3.8` |
| Resource Group | `Azure/avm-res-resources-resourcegroup/azurerm` | `0.2.2` |
| UAMI | `Azure/avm-res-managedidentity-userassignedidentity/azurerm` | `0.4.0` |
| Key Vault | `Azure/avm-res-keyvault-vault/azurerm` | `0.10.2` |
| Storage Account | `Azure/avm-res-storage-storageaccount/azurerm` | `0.6.7` |
| AI Search | `Azure/avm-res-search-searchservice/azurerm` | `0.2.0` |
| NSG (×4) | `Azure/avm-res-network-networksecuritygroup/azurerm` | `0.5.1` |
| Foundry Hub + Project | `azapi_resource` — no AVM module | `Microsoft.CognitiveServices/accounts@2025-04-01-preview` |
| GPT-4o deployment | `azapi_resource` — no AVM module | `Microsoft.CognitiveServices/accounts/deployments@2024-10-01` |

All versions are the latest stable published release on the Terraform public registry as of
2026-03-08, verified via `mcp_terraform_get_latest_module_version` and
`mcp_terraform_get_latest_provider_version`. Minor-version constraint (`~> X.Y`) is used per
Terraform pinning best practice.

---

## 8. Private DNS Zones

### Decision

All private DNS zones are **data sources** from the resource group named in
`var.private_dns_rg_name`. No new DNS zones are created (FR-015).

| Service | Zone | Variable name in data source |
| --- | --- | --- |
| AI Search | `privatelink.search.windows.net` | `data.azurerm_private_dns_zone.search` |
| Storage (blob) | `privatelink.blob.core.windows.net` | `data.azurerm_private_dns_zone.blob` |
| Key Vault | `privatelink.vaultcore.azure.net` | `data.azurerm_private_dns_zone.keyvault` |
| Microsoft AI Foundry Hub | `privatelink.cognitiveservices.azure.com` | `data.azurerm_private_dns_zone.cognitiveservices` |

All AVM module PEs specify `private_dns_zone_resource_ids` within their module call to register
DNS A records automatically (AVM default `private_endpoints_manage_dns_zone_group = true`).
The Foundry Hub PE DNS zone group is registered via a separate `azapi_resource` targeting
`Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2024-01-01` because the Pe is not
created through an AVM module.

**Removed zones** (vs. previous ML Hub design):

- `privatelink.openai.azure.com` — no longer needed; OpenAI endpoint is on the Foundry Hub account, accessible via `privatelink.cognitiveservices.azure.com`
- `privatelink.api.azureml.ms` — ML workspace specific; not applicable to CognitiveServices
- `privatelink.notebooks.azure.net` — ML workspace specific; not applicable to CognitiveServices

---

## 9. RBAC Architecture

### Assignment Matrix

| Role | Scope (resource) | Principal |
| --- | --- | --- |
| `Search Index Data Contributor` | AI Search instance | UAMI |
| `Search Service Contributor` | AI Search instance | UAMI |
| `Storage Blob Data Contributor` | Storage Account | UAMI |

All role assignments are wired through the `role_assignments` map on the respective AVM module.
No `azurerm_role_assignment` resources are created outside of AVM module calls.

**Removed from previous ML Hub design**: `Cognitive Services OpenAI User` on Azure OpenAI is
eliminated because there is no separate Azure OpenAI resource. The UAMI IS the identity of
the Foundry Hub account, so no cross-resource role is needed for model inferencing.

---

## 10. Storage Account Naming Uniqueness

### Decision

Use `resource "random_string" "storage_suffix"` (4 lowercase alphanumeric characters) appended
to the storage base-name segment to guarantee global uniqueness while keeping the name within
the 24-character limit. The `random_string` resource uses `keepers = {}` (empty) so it only
generates on first apply and is stable across re-runs.

### Rationale

FR-008 mandates pseudorandom suffix for globally unique resources. The `random` provider is
already required in `terraform.tf`. The base Storage name (without suffix) uses abbreviated
segments to stay within 24 chars after suffix is appended.

---

## 11. Storage Account Key Disablement

### Decision

Set `shared_access_key_enabled = false` on the Storage Account module. The AI Search indexer
data source will use UAMI-based authentication (managed identity client ID in the data source
credentials object). `storage_access_type = "identity"` is set on the ML workspace Hub.

### Rationale

FR-016 prohibits connection strings, API keys, and shared access signatures. Disabling the
storage account key at the resource level enforces this in the control plane.

---

## 12. AI Search SKU Naming

### Decision

Use `sku = "standard"` in `avm-res-search-searchservice`. The module's `sku` input maps
`"standard"` → Standard S1 (confirmed from module docs where default is `"standard"`).
Enable semantic ranking with `semantic_search_sku = "standard"`.

### Rationale

The spec requires Standard S1 for private endpoints, vector indexing, and semantic ranking
(FR-003). The module expresses the S1 tier as `sku = "standard"` (not `"standard1"`), which
is consistent with the azurerm provider naming convention for AI Search SKUs.
