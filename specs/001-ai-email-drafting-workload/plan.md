# Implementation Plan: 001-ai-email-drafting-workload

**Branch**: `001-ai-email-drafting-workload` | **Spec**: [spec.md](./spec.md)  
**Input**: Feature specification from `specs/001-ai-email-drafting-workload/spec.md`

## Summary

Provision the complete infrastructure for an AI Email Response Drafting application on Azure
using Terraform and Azure Verified Modules (AVM) exclusively where coverage exists, and
`azapi_resource` for resources without an AVM module. The deployment provisions a
**Microsoft AI Foundry Hub** (`Microsoft.CognitiveServices/accounts` kind=`AIServices`) via
`azapi_resource`, with a **Foundry Project** as a sub-resource, and a **GPT-4o model
deployment** directly on the Foundry Hub account. Supporting services include an
**Azure AI Search S1** service with semantic ranking, a **Storage Account** (ZRS), and an
**Azure Key Vault** (standard SKU). Every service is reachable only via private endpoints
inside the customer VNet. A single user-assigned managed identity is used for all cross-service
RBAC. All resources land in `canadacentral`.

Full research findings, module decisions, and the RBAC matrix are documented in
[research.md](./research.md). The full data model (locals, module calls, `azapi_resource`
blocks, dependency chain) is in [data-model.md](./data-model.md). The operator interface
contract is in [contracts/terraform-interface.md](./contracts/terraform-interface.md).

---

## Technical Context

**Language/Version**: HCL — Terraform `~> 1.11`  
**Primary Dependencies**:

- Provider `hashicorp/azurerm` `~> 4.63`
- Provider `azure/azapi` `~> 2.8`
- Provider `hashicorp/random` `~> 3.8`
- 8 AVM resource modules (versions pinned — see table below)

| Module | Version |
| --- | --- |
| `Azure/avm-res-resources-resourcegroup/azurerm` | `0.2.2` |
| `Azure/avm-res-managedidentity-userassignedidentity/azurerm` | `0.4.0` |
| `Azure/avm-res-keyvault-vault/azurerm` | `0.10.2` |
| `Azure/avm-res-storage-storageaccount/azurerm` | `0.6.7` |
| `Azure/avm-res-search-searchservice/azurerm` | `0.2.0` |
| `Azure/avm-res-network-networksecuritygroup/azurerm` | `0.5.1` |
| Microsoft AI Foundry Hub + Project | `azapi_resource` — `Microsoft.CognitiveServices/accounts@2025-04-01-preview` |
| GPT-4o deployment | `azapi_resource` — `Microsoft.CognitiveServices/accounts/deployments@2024-10-01` |

**Storage**: Azure Storage Account (ZRS), Azure Key Vault (secrets at rest), AI Search index  
**Testing**: `terraform fmt -check` → `terraform validate` → `terraform plan` (pre-apply gates)  
**Target Platform**: Azure — region `canadacentral`  
**Project Type**: Infrastructure-as-Code root module  
**Performance Goals**: N/A (infrastructure provisioning)  
**Constraints**:

- AVM modules only; `azapi_resource` for resources without AVM coverage (Microsoft AI Foundry
  Hub, Foundry Project, GPT-4o model deployment, PE for Foundry Hub, subnets, AI Search
  data-plane, Foundry Hub diagnostic settings)
- No hard-coded subscription IDs, resource IDs, or credentials — operator supplies all via
  `terraform.tfvars`
- Six-segment kebab-case naming: `<workload>-<env>-<type>-<region>-<instance>-<org>`
- Storage Account name uses `random_string` suffix to satisfy 24-char, no-hyphen constraint
- Microsoft AI Foundry Hub uses `publicNetworkAccess = "Disabled"` + private endpoint for
  isolation (`AllowOnlyApprovedOutbound` is ML-workspace-specific; equivalent isolation achieved
  via PE + network ACLs on CognitiveServices resource)
- All services must have a diagnostic settings block pointing to operator-supplied LAW

**Scale/Scope**: Single resource group, single deployment environment

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Gate | Status | Notes |
| --- | --- | --- | --- |
| **P1 — AVM-only IaC** | No raw `azurerm_*` resources; `azapi_resource` allowed only where no AVM module exists | ✅ PASS | Subnets, Foundry Hub, Foundry Project, model deployment, Foundry PE, Foundry diagnostic settings use `azapi_resource`; all other resources use AVM |
| **P2 — Security by Default** | All services: public access off, private endpoints, RBAC only, no shared keys, diagnostic logs | ✅ PASS | `publicNetworkAccess = "Disabled"` + `networkAcls.defaultAction = "Deny"` on Foundry Hub; `public_network_access_enabled = false` on all AVM-managed services; `shared_access_key_enabled = false` on Storage; UAMI for all cross-service auth; LAW diagnostics on all resources |
| **P3 — Validate Before Deploy** | `terraform fmt -check`, `terraform validate`, `terraform plan` must run and pass before every `apply` | ✅ PASS | Enforced in `quickstart.md` Steps 4–5; constitution requirement documented |
| **P4 — Canada Central** | `location = "canadacentral"` on all resources | ✅ PASS | `var.location` defaults to `"canadacentral"`; validated by a `precondition` in `variables.tf` |
| **P5 — Managed Identity** | No service principals or connection-string auth; all RBAC uses UAMI | ✅ PASS | Single UAMI holds all four role assignments; OpenAI connected to Hub via `aml_workspace.resource_id` |

**No violations. No complexity tracking entry required.**

---

## Project Structure

### Documentation (this feature)

```text
specs/001-ai-email-drafting-workload/
├── plan.md                          # This file
├── research.md                      # Phase 0 — module decisions, architecture, RBAC
├── data-model.md                    # Phase 1 — modules, azapi blocks, dependency chain
├── quickstart.md                    # Phase 1 — operator deploy guide
├── contracts/
│   └── terraform-interface.md       # Phase 1 — input/output/prerequisite contracts
└── tasks.md                         # Phase 2 — created by /speckit.tasks, not this command
```

### Source Code (repository root)

```text
src/
├── main.tf           # All module calls and azapi_resource blocks (no inline locals/vars)
├── variables.tf      # All variable declarations with descriptions and validations
├── outputs.tf        # 9 resource ID outputs; no secrets
├── terraform.tf      # required_version + required_providers; provider version pins
└── terraform.tfvars  # Operator-supplied values — NOT committed to VCS
```

**Structure Decision**: Single flat root module. No child modules are introduced because
the workload is a single-environment infrastructure deployment with no reuse requirement.
All resource definitions live in `main.tf` with `locals{}` blocks for naming and dependency
consolidation. `azapi_resource` blocks for subnets, model deployments, AI Search data-plane,
and workspace connections are co-located in `main.tf` alongside their AVM module counterparts.
