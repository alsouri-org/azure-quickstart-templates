---
name: contribution-guide
description: Guide for creating or modifying Azure Quickstart Templates. Use this when creating a new template, adding files to an existing template, or when asked about contribution requirements, folder structure, metadata, README, or parameter file placeholders.
---

# Contribution Guide for Azure Quickstart Templates

## Folder Structure & Location

Place every template in its own subfolder under the appropriate top-level folder:

| Folder | Purpose |
|---|---|
| `quickstarts/` | Quickly provision a single resource or simple set of resources |
| `application-workloads/` | Full application workloads (production-ready after deploy) |
| `demos/` | Demonstrate a specific Azure platform capability |
| `modules/` | Reusable Bicep/ARM modules shared across samples |
| `managementgroup-deployments/` | Templates deployed at management group scope |
| `subscription-deployments/` | Templates deployed at subscription scope |
| `tenant-deployments/` | Templates deployed at tenant scope |

Rules:
- All folder and file names must be **lowercase** (exception: `README.md`)
- Keep folder names short and descriptive (e.g. `vm-from-user-image`, `storage-static-website`)
- One PR per template — never update multiple templates in a single PR

## Required Files Per Template

| File | Notes |
|---|---|
| `main.bicep` | Preferred entry point. Required unless using JSON only. |
| `azuredeploy.json` | Required only when not using Bicep. **Do NOT include if `main.bicep` exists** — built automatically on merge. |
| `azuredeploy.parameters.json` | Always required, even for Bicep templates |
| `metadata.json` | Always required |
| `README.md` | Always required |

Optional files:
- `azuredeploy.parameters.us.json` — for US Government cloud
- `nestedtemplates/` — for JSON nested templates
- `modules/` — for Bicep modules
- `scripts/` — for configuration scripts
- `prereqs/` — for pre-requisite templates (see below)

## metadata.json

Must follow this structure exactly:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "itemDisplayName": "Short display name (max 60 chars)",
  "description": "Long description, up to 1000 chars",
  "summary": "Short summary, up to 200 chars",
  "githubUsername": "your-github-username",
  "dateUpdated": "YYYY-MM-DD",
  "type": "QuickStart"
}
```

- `itemDisplayName`: short, meaningful label shown in the index
- `summary`: one sentence on what the template does
- `description`: fuller explanation for the index page
- `dateUpdated`: update this whenever the template changes
- Do not change `githubUsername` unless taking ownership

## README.md Structure

Every README.md must include:
1. **Deploy to Azure** button (and Visualize button)
2. Description of what the template deploys
3. Tags (comma-separated, in backticks)
4. Parameter table
5. Optional: prerequisites, usage notes

See `1-CONTRIBUTION-GUIDE/sample-README.md` for the button code and full template.

## azuredeploy.parameters.json — CI Placeholders

Use these special placeholder values in parameters files so CI can auto-generate valid values:

| Placeholder | Use for |
|---|---|
| `GEN-UNIQUE` | Globally unique names (18 chars alphanumeric) |
| `GEN-UNIQUE-N` | Unique name of length N (3–32) |
| `GEN-PASSWORD` | Passwords |
| `GEN-SSH-PUB-KEY` | SSH public keys |
| `GEN-GUID` | GUIDs |
| `GEN-VNET-NAME` | Existing VNet name |
| `GEN-KEYVAULT-NAME` | Existing Key Vault name |

Do NOT use real values, `changeme`, or empty strings in the parameters file.

## Parameters Best Practices

- Always include `location` parameter with default `resourceGroup().location` — never add `allowedValues` to it
- Never hardcode resource names — always parameterize
- Never set default values for usernames, passwords, or `@secure()` parameters
- Use `uniqueString(resourceGroup().id)` in defaults for globally unique names
- Use `@allowed()` only for exclusive/constrained lists — not for inclusive lists like all VM SKUs
- All parameters must have a `@description()` decorator (Bicep) or `metadata.description` (JSON)

## Naming Conventions

- **camelCase** for all symbols: parameters, variables, resources, outputs
- Parameters named after the property they set: `storageAccountName`, `vmSize`, `location`
- Resource symbolic names: short and logical — `storage`, `vm`, `vnet`, `nic`

## Pre-requisites

If the template needs pre-existing resources (e.g. an existing VNet):
1. Create a `prereqs/` subfolder
2. Add `prereq.azuredeploy.json` or `prereq.main.bicep` and `prereq.azuredeploy.parameters.json`
3. Output the values needed by the main template from the prereq template
4. Reference prereq outputs in the main parameters file as `GET-PREREQ-<OutputName>`

## Deployment Artifacts

If the template needs scripts or nested templates:
- Store nested JSON templates in `nestedtemplates/`
- Store Bicep modules in `modules/`
- Store scripts in `scripts/`
- Use `_artifactsLocation` and `_artifactsLocationSasToken` parameters for artifact URIs
- Always construct URIs using the `uri()` function — never hardcode raw GitHub URLs

## Outputs

- Include outputs for any values useful after deployment (endpoints, IPs, FQDNs)
- **Never** output secrets, passwords, or account keys

## Validation Before Committing

```bash
# Validate ARM/Bicep template
az deployment group validate \
  --resource-group <rg> \
  --template-file main.bicep \
  --parameters azuredeploy.parameters.json

# Lint Bicep
az bicep lint --file main.bicep

# Build Bicep to ARM JSON (do NOT commit the output)
az bicep build --file main.bicep
```
