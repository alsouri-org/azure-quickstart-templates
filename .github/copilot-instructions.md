# Copilot Instructions for Azure Quickstart Templates

This repository contains Azure Resource Manager (ARM) and Bicep templates contributed by the community.

## Repository Structure

Templates must be placed in the appropriate top-level folder based on their purpose:

| Folder | Purpose |
|---|---|
| `quickstarts/` | Quickly provision a single resource or simple set of resources |
| `application-workloads/` | Full application workloads (production-ready after deploy) |
| `demos/` | Demonstrate a specific Azure platform capability |
| `modules/` | Reusable Bicep/ARM modules shared across samples |
| `managementgroup-deployments/` | Templates deployed at management group scope |
| `subscription-deployments/` | Templates deployed at subscription scope |
| `tenant-deployments/` | Templates deployed at tenant scope |

Each template lives in its own subfolder. Folder and file names must be **lowercase** (exception: `README.md`).

## Required Files Per Template

Every template folder must contain:

| File | Notes |
|---|---|
| `main.bicep` | Preferred. Required if not using `azuredeploy.json` |
| `azuredeploy.json` | Required only if not using Bicep. **Do NOT include if `main.bicep` exists** — it is built automatically on merge |
| `azuredeploy.parameters.json` | Always required |
| `metadata.json` | Always required — used for the searchable index |
| `README.md` | Always required |

Optional: `azuredeploy.parameters.us.json` for US Government cloud, `nestedtemplates/` for JSON nested templates, `modules/` for Bicep modules, `scripts/` for configuration scripts.

## Bicep Preference

**Prefer Bicep over ARM JSON for all new or updated templates.**

- Entry point file must be named `main.bicep`
- Do NOT submit `azuredeploy.json` alongside `main.bicep` — it is auto-generated on merge
- Parameters file is always `azuredeploy.parameters.json` (even for Bicep templates)
- Top-level element order: `targetScope` → `param` → `var` → `resource`/`module` → `output`
- Add `@description()` decorator to every parameter; place it first among all decorators
- Use `@allowed()` for constrained values; do NOT use it for inclusive lists (e.g. all VM SKUs)

## Parameters

- Always include a `location` parameter with default `resourceGroup().location` — no `allowedValues`
- Never hardcode resource names — always use `parameters()` or `variables()`
- Never set default values for usernames, passwords, or any secure parameter
- Credentials must always be parameterized (`@secure()` in Bicep, `"type": "securestring"` in JSON)
- Use `uniqueString(resourceGroup().id)` in defaults where globally unique names are needed

## Naming Conventions

- Use **camelCase** for all symbol names (parameters, variables, resources, outputs)
- Name parameters after the specific property they set: `storageAccountName`, not `name`
- Resource symbolic names should be short and logical: `storage`, `vm`, `vnet`

## metadata.json Schema

```json
{
  "itemDisplayName": "Short display name (max 60 chars)",
  "description": "One sentence description",
  "summary": "One sentence summary",
  "githubUsername": "your-github-username",
  "dateUpdated": "YYYY-MM-DD"
}
```

## Validation

Before committing any template changes, validate with:

```bash
az deployment group validate \
  --resource-group <rg> \
  --template-file main.bicep \
  --parameters azuredeploy.parameters.json
```

For Bicep, also run:

```bash
az bicep lint --file main.bicep
```

## Pull Requests

- One PR per template — never update multiple templates in a single PR
- A single PR should only touch files within one template folder (plus root-level files if needed)
