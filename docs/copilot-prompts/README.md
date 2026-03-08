# Copilot Prompts

This file contains the GitHub Copilot prompts used, including those to build the specifications with [Spec Kit](https://github.github.com/spec-kit/).

## Constitution

/speckit.constitution Fill the constitution with the typical requirements of an AI Azure workload (needed to be retained for compliance reasons; no high-availability requirements; no disaster recovery requirements; no scalability requirements), defined as infrastructure-as-code, in Terraform language, built only with Azure Verified Modules (AVM). Always use Terraform, and never use custom scripts. Security and reliability best practices must be followed under all circumstances. Before running a deployment, always run a validation. Deploy everything to the Canada Central datacenter region.

## Specification

/speckit.specify Create specification, called "01-ai-email-drafting-workload" for an AI email response drafting app that has a Power Platform component. The Power Platform component is out of scope for this specification.

The app needs an Azure AI Foundry instance supported by Azure AI Search for vector search. The knowledge base in Azure AI Foundry will get data from an Azure Storage Account with a blob container. The blobs in the storage container must be indexed by AI Search.

The Azure AI Foundry, AI Search, and Storage Account must all use private endpoints. The DNS configuration for the private endpoints will use already
existing Azure private DNS zones for the privatelink zones. Get the resource group of those DNS zones from `terraform.tfvars`.

The Azure services must call each other using managed identity authentication only. Create the necessary role assignments in Terraform.

The AI Foundry will use an OpenAI large language model (LLM).

Always rely on values from the `terraform.tfvars` file only. Include rich comments in both the `main.tf` and `terraform.tfvars` files to explain the purpose of each resource, module, and variable.

When a decision needs to be made on availability zones, prefer zone redundancy. If a service does not support zone redundant deployments, always choose a number between 1 and 3 (never choose -1, that explicitly disables this feature).

Create all Azure resources in a single resource group, standing for a production environment. Do not create any additional environments (such as dev, test, staging, etc.).

Read the documentation (readme.md file) of each module you need to use to find out what variables and complex variable objects you can use. Do not guess the allowed variables.

Configure diagnostic logging for all Azure services to a centralized, already existing, Log Analytics Workspace. Obtain the resource ID of the workspace from `terraform.tfvars`.

The Azure virtual network has already been provisioned, but subnets must still be created. Obtain the resource ID of the existing virtual network from `terraform.tfvars`.

The Azure resource naming convention must include the following six segments, in order: workload name, environment, resource type (abbreviation), region, instance number (two digits, starting at `01`), and a short organization identifier. The values of these segments must be obtained from `terraform.tfvars`. Resource type specific character and length limitations must be respected. Random character should only be added to resources that must be globally unique, like storage accounts. All resource names should be kebab case unless the hyphen is not supported for that resource type. Obtain the resource type abbreviations from [https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations].

---

/speckit.specify Update the specification to use a single user-assigned managed identity for all services.

## Clarify

/speckit.clarify

## Plan

/speckit.plan Create a detailed plan for the spec. Build with the latest version of Terraform and the latest available version of each Azure Verified Module. Use the Terraform MCP server ("io.github.hashicorp/terraform-mcp-server") to find out what the latest version of each module is. Install and configure this MCP server as needed. Do NOT use the "Bicep/list_avm_metadata" MCP tool! Only include direct resource references in the Terraform solution template (root module) if no related AVM resource modules are available. If there is no Azure Verified Module available, then use `azapi` provider resources, never use the `azurerm` provider directly. Always use module interfaces for diagnostic settings, role assignments, resource locks, tags, managed identities, private endpoints, customer manged keys, etc., always use the related "interface" built-in to each resource module when available. Do not create and reference local modules, or any other Terraform files. If a subset of the deployments fail, don't delete anything, just attempt redeploying the whole solution after fixing any bugs. Follow IaC best practices: define everything in a single root module using the standard module files of `main.tf`, `variables.tf`, `outputs.tf`, `terraform.tf`, and `terraform.tfvars`. Only use explicit dependencies with the `depends_on` meta-argument when it's not possible to otherwise determine the order of deployment.

The Azure subscription ID will always be supplied via az cli, it must not be exposed as a variable.

Terraform solution template (root module) must validate without warnings or errors using the latest stable Terraform CLI version. Generate a warning when the latest version of an AVM module is not used. Before validating the solution template (root module) or attempting the first deployment, always fix all warnings or errors related to the AVM module versioning by updating to the latest available version of each module.

Always use snake case for Terraform HCL resource names, module names, variable names, output names, map keys, etc. Never shorten names, always use the full name. E.g. `network_security_group` instead of `nsg`, etc.

Ephemeral resources and write-only attributes should be used for passwords.

---

/speckit.plan You are creating an Azure AI Hub which is not necessary. We need a Microsoft Foundry instance. There is no AVM module for Microsoft Foundry at this time. Update the plan to use the `azapi` to create the Foundry resource while maintaining all requirements.

TODO: /speckit.plan Reconsider using the Azure AI Foundry Pattern module. It does support private deployments.

## Specification Update

TODO: /speckit.specify Update the specification to use Microsoft Foundry and not "Azure AI Foundry project" or "hub," which created confusion in the plan phase because it would create an Machine Learning Workspace / AI studio.
