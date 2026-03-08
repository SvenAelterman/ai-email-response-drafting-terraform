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
