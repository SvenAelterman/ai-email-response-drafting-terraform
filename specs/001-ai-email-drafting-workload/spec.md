# Feature Specification: AI Email Response Drafting Workload

**Feature Branch**: `001-ai-email-drafting-workload`  
**Created**: 2026-03-08  
**Status**: Draft  
**Last Updated**: 2026-03-08 — Updated to use a single user-assigned managed identity for all services.  
**Input**: User description: "Create specification for an AI email response drafting app that has a Power Platform component. The Power Platform component is out of scope. The app needs Azure AI Foundry, Azure AI Search, Azure Storage Account, private endpoints, managed identity, AVM modules, Terraform-only, Canada Central region."

## Scope Boundary

**In scope**: Terraform infrastructure code for the Azure AI workload — Azure AI Foundry, Azure AI Search, Azure Storage Account, Virtual Network subnets, private endpoints, DNS integration, managed identity role assignments, diagnostic logging, and resource naming.

**Out of scope**: Power Platform components (connectors, flows, apps, Copilot Studio agents), application-layer code, CI/CD pipeline definitions, and any non-Azure infrastructure.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Deploy core AI services infrastructure (Priority: P1)

An infrastructure engineer runs `terraform apply` against a configured `terraform.tfvars`
file and has all required Azure AI services (Azure AI Foundry project, Azure AI Search,
Azure Storage Account) provisioned in a single production resource group in Canada Central,
with resources named according to the six-segment naming convention.

**Why this priority**: This is the foundation of the workload. Nothing else can be tested
or demonstrated without the core services being deployed.

**Independent Test**: Can be fully tested by running `terraform apply` with a valid
`terraform.tfvars`, then verifying each resource exists in the Azure portal or via
`terraform show`. Delivers a fully deployed AI stack with correct naming.

**Acceptance Scenarios**:

1. **Given** a valid `terraform.tfvars` with all required segment values, **When** `terraform apply` runs, **Then** an Azure AI Foundry hub and project, an Azure AI Search instance, and an Azure Storage Account with a blob container are created in `canadacentral` inside a single resource group.
2. **Given** the deployed resources, **When** their names are inspected, **Then** every name follows the pattern `<workload>-<env>-<type-abbrev>-<region>-<instance>-<org>` (kebab-case, with resource-type character limits respected; globally unique resources include random characters).
3. **Given** the workload's `terraform.tfvars` is the only configuration file, **When** any configurable value (region, instance number, org identifier, etc.) is changed there, **Then** `terraform plan` reflects the change without any edits to `main.tf`.

---

### User Story 2 — Secure private networking (Priority: P2)

An infrastructure engineer verifies that Azure AI Foundry, AI Search, and the Storage
Account are accessible only through private endpoints on the existing virtual network,
that public network access is disabled on all three, and that DNS resolution is handled
by the organization's existing Azure private DNS zones.

**Why this priority**: Security is non-negotiable per the constitution. Private
connectivity must be in place before the workload can be approved for production use.

**Independent Test**: Can be fully tested by confirming that public endpoint access is
blocked (HTTP 403/network unreachable from outside the VNet), that private endpoints exist
in the correct subnets, and that DNS resolution from within the VNet returns private IPs.

**Acceptance Scenarios**:

1. **Given** the deployed resources, **When** public network access is tested from outside the VNet, **Then** all three services (AI Foundry, AI Search, Storage) refuse the connection or return a network-level error.
2. **Given** the deployed private endpoints, **When** DNS is queried from within the VNet for the service FQDNs, **Then** the resolution returns private IP addresses (not public ones).
3. **Given** the `terraform.tfvars` contains the existing private DNS zone resource group name, **When** Terraform applies, **Then** private DNS A records are registered in the correct existing private DNS zones (no new DNS zones are created).
4. **Given** the existing virtual network resource ID in `terraform.tfvars`, **When** Terraform applies, **Then** the required subnets are created inside that VNet and private endpoints are attached to those subnets.

---

### User Story 3 — Single user-assigned managed identity authentication and role assignments (Priority: P3)

An infrastructure engineer verifies that all service-to-service communication (AI Foundry
to AI Search, AI Foundry to Storage, AI Search to Storage) relies exclusively on a single
user-assigned managed identity shared across all three services, with Azure RBAC role
assignments granting the minimum required permissions. No connection strings, API keys, or
shared access signatures are in use.

**Why this priority**: Passwordless authentication using a single, centrally managed
identity is a hard security requirement and simplifies ongoing access governance. Without
it, the workload cannot pass a compliance review.

**Independent Test**: Can be fully tested by running `terraform show` to confirm a single
user-assigned managed identity resource exists, is assigned to all three services, and
the corresponding RBAC role assignments are present; and by verifying that no
keys/connection strings are exposed in Terraform state or outputs.

**Acceptance Scenarios**:

1. **Given** the deployed resources, **When** their identity settings are inspected, **Then** exactly one user-assigned managed identity exists and is assigned to Azure AI Foundry, Azure AI Search, and the Azure Storage Account — no system-assigned identities and no separate per-service identities are used.
2. **Given** the deployed resources, **When** RBAC role assignments are listed for the single user-assigned managed identity, **Then** it holds the minimum required roles on AI Search and Storage to allow AI Foundry to perform inference and indexing, and AI Search to read blobs.
3. **Given** the Terraform state and outputs, **When** they are inspected for secrets, **Then** no storage account keys, AI Search admin keys, or AI Foundry connection strings appear in any output or state value.

---

### User Story 4 — Diagnostic logging to centralized Log Analytics Workspace (Priority: P4)

An infrastructure engineer verifies that diagnostic settings are configured on all Azure
services, forwarding logs and metrics to the organization's existing Log Analytics
Workspace whose resource ID is supplied via `terraform.tfvars`.

**Why this priority**: Observability is required for compliance and operational support.
Without it, audit requirements cannot be satisfied.

**Independent Test**: Can be fully tested by running `terraform show` to confirm
diagnostic setting resources exist on each service, and by verifying that log records
appear in the Log Analytics Workspace after a short activity period.

**Acceptance Scenarios**:

1. **Given** the Log Analytics Workspace resource ID in `terraform.tfvars`, **When** Terraform applies, **Then** diagnostic settings resources are created for AI Foundry, AI Search, and the Storage Account, all pointing to that workspace.
2. **Given** the diagnostic settings, **When** their category configurations are inspected, **Then** all available log and metric categories are enabled (or the maximum meaningful set, per the service's capabilities).

---

### User Story 5 — Validate before deploy (Priority: P5)

Anyone running infrastructure changes first executes `terraform validate` and reviews a
clean `terraform plan` output before issuing `terraform apply`, ensuring no accidental
or unexpected resource changes are introduced.

**Why this priority**: The constitution mandates validate-before-deploy as non-negotiable.
This story formalizes that workflow requirement.

**Independent Test**: Can be fully tested by running `terraform fmt -check`,
`terraform validate`, and `terraform plan` independently, verifying all pass with zero
errors and a plan that matches the expected resource count.

**Acceptance Scenarios**:

1. **Given** the repository is in a clean state, **When** `terraform fmt -check` is run, **Then** it exits with code 0 and reports no formatting issues.
2. **Given** a configured backend and `terraform.tfvars`, **When** `terraform validate` is run after `terraform init`, **Then** it exits with code 0 and reports "Success! The configuration is valid."
3. **Given** a valid plan, **When** the plan output is reviewed, **Then** no resource outside the expected set (resource group, AI Foundry hub, AI Foundry project, AI Search, Storage Account, subnets, private endpoints, DNS records, role assignments, diagnostic settings) appears in the diff.

---

### Edge Cases

- What happens when the existing virtual network does not have available IP address space for the required subnets? → Terraform MUST fail with a clear error referencing the CIDR range conflict; the operator must update `terraform.tfvars` with a valid CIDR range.
- What happens when the specified private DNS zone resource group does not contain the expected zones? → Terraform MUST fail with a descriptive error identifying the missing zone(s) so the operator can remediate.
- What happens when the Log Analytics Workspace resource ID in `terraform.tfvars` is invalid or the workspace does not exist? → Terraform `plan` or `apply` MUST fail with a clear resource-not-found error rather than silently skipping diagnostic settings.
- What happens when a resource name exceeds the maximum length for its type due to long segment values? → The naming logic MUST truncate or abbreviate predictably and be documented in a comment in `main.tf`; the generated name MUST remain unique and within limits.
- What happens when an AVM does not support a required configuration option (e.g., zone redundancy for a particular SKU)? → The module variable for availability zones MUST be set to an explicit value between 1 and 3 (not -1), and this decision MUST be documented in a comment.

---

## Requirements *(mandatory)*

### Functional Requirements

#### Infrastructure provisioning

- **FR-001**: The infrastructure MUST be defined exclusively in Terraform (`.tf` files) using only Azure Verified Modules (AVM) sourced from the Terraform public registry. No ARM, Bicep, or custom provisioning scripts are permitted.
- **FR-002**: Terraform MUST provision an Azure AI Foundry hub and an Azure AI Foundry project within a single production resource group in the `canadacentral` region.
- **FR-003**: Terraform MUST provision an Azure AI Search instance in the same resource group configured for vector-search capabilities.
- **FR-004**: Terraform MUST provision an Azure Storage Account with a blob container in the same resource group; this container serves as the knowledge-base data source for AI Foundry.
- **FR-005**: Terraform MUST provision the required subnets inside the existing virtual network (resource ID obtained from `terraform.tfvars`) for private endpoint attachment.
- **FR-006**: Terraform MUST configure an OpenAI large language model (LLM) deployment on the Azure AI Foundry hub.

#### Naming and configuration

- **FR-007**: Every Azure resource name MUST follow the six-segment naming convention: `<workload_name>-<environment>-<resource_type_abbreviation>-<region>-<instance_number>-<org_identifier>`, using kebab-case. Segment values MUST be obtained exclusively from `terraform.tfvars`.
- **FR-008**: Resource type abbreviations MUST be sourced from the [Microsoft Cloud Adoption Framework naming conventions](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations). Resource-type character and length limitations MUST be respected; globally unique resources (e.g., Storage Account) MUST append pseudorandom characters to ensure uniqueness.
- **FR-009**: All configurable values (region, workload name, environment, instance number, org identifier, existing resource IDs, CIDR ranges, LLM model name, etc.) MUST reside in `terraform.tfvars` only. No values may be hard-coded in `main.tf`.
- **FR-010**: Both `main.tf` and `terraform.tfvars` MUST include rich inline comments explaining the purpose of each resource, module block, and variable.
- **FR-011**: When a decision is required on availability zones, zone redundancy MUST be preferred. If zone redundancy is not supported, a value between 1 and 3 MUST be chosen (never -1). This decision MUST be documented in a comment next to the relevant variable.
- **FR-012**: All resources MUST be created in a single resource group representing a production environment. No additional environments (dev, test, staging) are provisioned.
- **FR-013**: Before invoking any AVM module, the module's `README.md` documentation MUST be consulted to identify supported variables and complex variable objects. Variables MUST NOT be guessed.

#### Security and networking

- **FR-014**: Azure AI Foundry, Azure AI Search, and the Azure Storage Account MUST each have a private endpoint configured. Public network access MUST be disabled on all three services.
- **FR-015**: Private endpoint DNS registration MUST use the organization's existing Azure private DNS zones. The resource group containing those zones MUST be obtained from `terraform.tfvars`. No new private DNS zones are to be created.
- **FR-016**: Terraform MUST provision exactly one user-assigned managed identity and assign it to Azure AI Foundry, Azure AI Search, and the Azure Storage Account. No system-assigned identities are to be used. Connection strings, API keys, and shared access signatures MUST NOT be used for any service-to-service communication.
- **FR-017**: Terraform MUST create all Azure RBAC role assignments on the single user-assigned managed identity that are necessary for AI Foundry to access AI Search and Storage, and for AI Search to read and index blobs from the Storage Account.

#### Observability

- **FR-018**: Diagnostic settings MUST be configured for Azure AI Foundry, Azure AI Search, and the Azure Storage Account, forwarding all available log and metric categories to an existing, centralized Log Analytics Workspace. The workspace resource ID MUST be obtained from `terraform.tfvars`.
- **FR-019**: Data and audit logs MUST be retained for a minimum of 90 days in the Log Analytics Workspace (or longer if required by applicable compliance obligations).

#### Deployment workflow

- **FR-020**: `terraform fmt -check` followed by `terraform validate` MUST pass before `terraform apply` is invoked. This sequence MUST never be skipped.

### Key Entities

- **Azure AI Foundry Hub**: The central AI platform resource. Hosts the OpenAI LLM deployment, connects to AI Search as the vector-search back-end, and uses the Storage Account as the knowledge-base data source.
- **Azure AI Foundry Project**: A logical workspace within the Foundry Hub where the email-drafting knowledge base and model deployments are organized.
- **Azure AI Search Instance**: Provides vector-search indexing and retrieval. Indexes blobs from the Storage Account and is queried by AI Foundry at inference time.
- **Azure Storage Account / Blob Container**: Stores the source documents used to build the AI knowledge base. AI Search indexes blobs from a dedicated container.
- **Private Endpoints**: Network interface resources that bind each of the three services to a subnet on the existing virtual network. DNS A records are registered in the existing private DNS zones.
- **User-Assigned Managed Identity**: A single user-assigned managed identity resource provisioned by Terraform and assigned to all three services (AI Foundry, AI Search, Storage Account). All RBAC role assignments for service-to-service access are made against this one identity, enabling centralized access governance and lifecycle management.
- **Subnets**: Subdivisions of the existing virtual network created by Terraform to host the private endpoints. CIDR ranges are specified in `terraform.tfvars`.
- **Log Analytics Workspace** *(pre-existing)*: Centralized observability target. Referenced by resource ID from `terraform.tfvars`; not created by this Terraform configuration.
- **Private DNS Zones** *(pre-existing)*: Authoritative DNS zones for `privatelink.*` domains. Referenced by resource group from `terraform.tfvars`; not created by this Terraform configuration.
- **Virtual Network** *(pre-existing)*: The hub or spoke VNet into which subnets are injected. Referenced by resource ID from `terraform.tfvars`; not created by this Terraform configuration.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: `terraform validate` completes with exit code 0 and zero errors on a fresh `terraform init`.
- **SC-002**: `terraform plan` produces a plan containing only the expected resource types with no unexpected additions, changes, or deletions.
- **SC-003**: `terraform apply` completes successfully (exit code 0) on a properly configured environment without manual intervention beyond approving the plan.
- **SC-004**: All deployed resources are visible in the Azure portal in the `canadacentral` region within the single production resource group and carry names that match the six-segment naming convention.
- **SC-005**: All public endpoint access attempts to AI Foundry, AI Search, and Storage Account fail with a network-level error, confirming private-only access is enforced.
- **SC-006**: DNS queries for the private endpoint FQDNs, issued from a host inside the virtual network, resolve to private IP addresses.
- **SC-007**: `terraform show` and the state file contain no storage account keys, AI Search admin keys, or service connection strings in plaintext.
- **SC-008**: Diagnostic log entries from AI Foundry, AI Search, and the Storage Account are visible in the centralized Log Analytics Workspace within 15 minutes of activity.
- **SC-009**: Exactly one user-assigned managed identity is deployed; it is assigned to all three services; and all required RBAC role assignments for service-to-service access are present and verifiable without any additional manual configuration.
- **SC-010**: `terraform fmt -check` exits with code 0, confirming all `.tf` files are consistently formatted.

---

## Assumptions

- The organization's existing Azure private DNS zones for `privatelink.search.windows.net`, `privatelink.blob.core.windows.net`, `privatelink.cognitiveservices.azure.com`, and `privatelink.openai.azure.com` (and any other relevant privatelink zones for AI Foundry) already exist and are linked to the virtual network.
- The existing virtual network has sufficient available IP address space for the subnets required by the private endpoints.
- An existing Log Analytics Workspace is available and its resource ID can be provided in `terraform.tfvars`.
- The Terraform state backend is already configured (e.g., Azure Storage-based remote backend). Backend configuration is outside the scope of this specification.
- The OpenAI LLM model selected for deployment (e.g., `gpt-4o`) is available in the `canadacentral` region and quota has already been approved.
- Azure Verified Modules for all required resource types (AI Foundry, AI Search, Storage Account, subnets, private endpoints, role assignments) exist in the Terraform registry. If a module is absent for a resource, feature work is blocked pending a constitution amendment.
- A single production resource group is sufficient; no geo-replication, zone-redundant storage replication across regions, or multi-region active-active topology is needed.
- Compliance data-retention of 90 days is the minimum; the organization may require a longer period, which would be specified at implementation time in `terraform.tfvars`.
- A single user-assigned managed identity is used for all three services. If an AVM module for a specific service does not support user-assigned managed identities, feature work is blocked pending a constitution amendment.
