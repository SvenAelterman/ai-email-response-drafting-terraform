<!--
SYNC IMPACT REPORT
==================
Version change: (new) → 1.0.0
Modified principles: N/A (initial authoring from template)
Added sections: Core Principles, Deployment Constraints, Development Workflow, Governance
Removed sections: N/A
Templates reviewed:
  ✅ .specify/templates/plan-template.md — Constitution Check gate present; no changes required
  ✅ .specify/templates/spec-template.md — FR format compatible; no changes required
  ✅ .specify/templates/tasks-template.md — Phase structure compatible; no changes required
Follow-up TODOs: None — all placeholders resolved
-->

# AI Email Response Drafting Terraform Constitution

## Core Principles

### I. Terraform-Only (NON-NEGOTIABLE)

All infrastructure MUST be defined in Terraform (`.tf` files). The following are
strictly forbidden: ARM templates, Bicep files, Azure CLI provisioning scripts,
PowerShell/Bash provisioning scripts, and any out-of-band manual resource creation.
Every resource lifecycle — create, update, destroy — MUST be managed exclusively through
`terraform apply`. This ensures reproducibility, auditability, and CI/CD compatibility.

### II. Azure Verified Modules Only (NON-NEGOTIABLE)

Every Azure resource MUST be provisioned using an official Azure Verified Module (AVM)
sourced from the Terraform registry (`registry.terraform.io/Azure/...`). Writing custom
`resource` blocks that duplicate functionality already available as an AVM is forbidden.
If an AVM does not exist for a required resource, an issue MUST be raised before
proceeding — no workarounds using raw resource blocks are permitted without explicit
constitution amendment.

### III. Security by Default (NON-NEGOTIABLE)

Security best practices MUST be applied unconditionally on every resource:

- **Identity**: All service-to-service authentication MUST use system-assigned or
  user-assigned managed identities. Connection strings, API keys, and passwords MUST
  NOT be used unless the target service offers no managed identity support.
- **Network**: All resources that support private endpoints MUST have a private endpoint
  configured. Public network access MUST be disabled on every such resource.
- **Encryption**: TLS/HTTPS MUST be enforced for all data in transit. Azure-managed keys
  are acceptable for data at rest; customer-managed keys are optional unless mandated by
  compliance.
- **Authorization**: Azure RBAC MUST be used for access control. Shared-access signatures
  and account-level keys MUST be disabled where role-based alternatives exist.
- **Secrets**: Any value that cannot be stored in source control (credentials, SAS tokens,
  etc.) MUST be stored in Azure Key Vault and referenced dynamically — never hard-coded.

### IV. Validate Before Deploy (NON-NEGOTIABLE)

`terraform validate` MUST pass, followed by a successful `terraform plan`, before any
`terraform apply` is executed. No deployment gate may be bypassed. CI pipelines MUST
enforce this sequence: `fmt check → init → validate → plan → (manual approval) → apply`.
Plan output MUST be reviewed for unexpected resource changes before approval is granted.

### V. Reliability and Observability

Reliability best practices MUST be applied even though high availability, disaster
recovery, and horizontal scalability are explicitly out of scope for this workload:

- Diagnostic settings MUST be enabled on all Azure resources, forwarding logs and
  metrics to a Log Analytics workspace.
- Resource locks (`CanNotDelete`) SHOULD be applied to production resource groups to
  prevent accidental deletion.
- Health and availability alerts SHOULD be configured for critical AI services (Azure AI
  Foundry, Azure AI Search).
- Single-region deployment (Canada Central) is intentional; no geo-replication is required.
- Implement zone redundancy when supported by the Azure service.

## Deployment Constraints

This section records scope boundaries that are non-negotiable for the current workload.
Changing any boundary below requires a MAJOR version amendment to this constitution.

| Constraint | Value | Rationale |
|---|---|---|
| Target region | `canadacentral` | Compliance data-residency requirement |
| High availability | **Not required** | Workload is a demo/sample; SLA not mandated |
| Disaster recovery | **Not required** | No RTO/RPO objectives defined |
| Scalability | **Not required** | Fixed capacity; no autoscale configuration |
| Data retention | **Required** | Audit logs and AI interaction data MUST be retained per applicable compliance obligations; minimum retention period is 30 days unless a longer period is mandated |
| Module source | AVM only | Consistency, supportability, security posture |
| IaC language | Terraform only | Toolchain standardization |

## Development Workflow

1. **Branch**: Every change MUST be made on a feature branch and merged via pull request.
2. **Lint & Format**: `terraform fmt -check` MUST pass. All `.tf` files MUST be formatted
   with `terraform fmt` before committing.
3. **Validate**: `terraform validate` MUST pass locally before pushing.
4. **Plan Review**: A `terraform plan` output MUST be attached to the pull request for
   reviewer inspection.
5. **Approval**: At least one peer review is REQUIRED before merge.
6. **Apply**: `terraform apply` MUST only run after merge to the default branch and after
   plan re-validation in the CI pipeline.
7. **No Drift**: Manual changes to deployed resources are forbidden. Any drift detected by
   `terraform plan` MUST be resolved through code, not ad-hoc CLI commands.

## Governance

This constitution supersedes all other project practices, guidelines, and team conventions.
Any conflict between this document and another guideline is resolved in favor of this
constitution.

**Amendment procedure**:

- PATCH: Wording clarifications and typo fixes — single approver, no migration plan needed.
- MINOR: New principle or section added — two approvers, update Sync Impact Report.
- MAJOR: Principle removal, redefinition, or scope boundary change — two approvers,
  documented migration plan, and update of all dependent template files required.

**Compliance review**: Constitution compliance MUST be verified during every pull request
review. Reviewers MUST confirm that the Constitution Check gate in `plan.md` is satisfied
before approving.

**Version**: 1.0.0 | **Ratified**: 2026-03-08 | **Last Amended**: 2026-03-08
