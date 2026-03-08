# Copilot Prompts

This file contains the GitHub Copilot prompts used, including those to build the specifications with [Spec Kit](https://github.github.com/spec-kit/).

## Constitution

/speckit.constitution Fill the constitution with the typical requirements of an AI Azure workload (needed to be retained for compliance reasons; no high-availability requirements; no disaster recovery requirements; no scalability requirements), defined as infrastructure-as-code, in Terraform language, built only with Azure Verified Modules (AVM). Always use Terraform, and never use custom scripts. Security and reliability best practices must be followed under all circumstances. Before running a deployment, always run a validation. Deploy everything to the Canada Central datacenter region.
