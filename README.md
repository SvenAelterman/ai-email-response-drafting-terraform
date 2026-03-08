# ai-email-response-drafting-terraform

A Terraform repository to deploy the Azure components for the AI Email Response Drafting sample at [https://github.com/microsoft/SLG-Business-Applications/blob/main/demos/administration/AI-Email-Response-Drafting/readme.md].

## Spec-Driven Development

This project uses [Spec-Kit](https://github.com/github/spec-kit).

## Changes from reference

- Private networking with private endpoints, etc.
- Using managed identity.
- Removing deprecated GPT4 OpenAI On Your Data.

## Azure resources

The following Azure resources are deployed.

- Azure AI Search
- Azure AI Foundry
- Virtual Network

## Bootstrapping tools

```PowerShell
winget install -e -h -s winget --id astral-sh.uv
# Restart Terminal to update PATH
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```
