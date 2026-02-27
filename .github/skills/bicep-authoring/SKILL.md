---
name: bicep-authoring
description: Guide for authoring Bicep templates in this repo. Use this when creating or editing .bicep or main.bicep files, converting ARM JSON to Bicep, or when asked about Bicep syntax, decorators, modules, or linting.
---

# Bicep Authoring Guide for Azure Quickstart Templates

## File Naming

- The entry point file **must** be named `main.bicep`
- Do **NOT** submit `azuredeploy.json` alongside `main.bicep` — it is auto-generated on merge via `az bicep build`
- Parameters file is always `azuredeploy.parameters.json` (even for Bicep templates)
- Bicep modules go in the `modules/` subfolder of the template

## Top-Level Element Order

Always use this order in `main.bicep`:

```bicep
targetScope = '...'   // omit if resourceGroup (default)

@description('...')
param ...

var ...

resource existing ... // existing resource references grouped together

resource/module ...   // new resources and modules

output ...
```

## Parameters

Every parameter must have `@description()` as the **first** decorator:

```bicep
@description('Location for all resources.')
param location string = resourceGroup().location

@description('Name of the storage account.')
param storageAccountName string = 'storage${uniqueString(resourceGroup().id)}'

@description('The admin username for the VM.')
param adminUsername string

@secure()
@description('The admin password for the VM.')
param adminPassword string

@description('The SKU of the storage account.')
@allowed([
  'Standard_LRS'
  'Standard_GRS'
  'Standard_ZRS'
  'Premium_LRS'
])
param storageAccountSku string = 'Standard_LRS'
```

Rules for parameters:
- `location` must default to `resourceGroup().location` — never add `@allowed()` to it
- Never set a default for `@secure()` parameters (passwords, keys, secrets)
- Use `uniqueString(resourceGroup().id)` to generate unique defaults for globally unique names
- Use `@allowed()` only for constrained/exclusive lists — not inclusive ones (e.g. not all VM SKUs)
- Use `@minValue()`/`@maxValue()` for numeric ranges
- Place a blank line before and after each parameter block

## Variables

- Use variables for values used multiple times or for complex expressions
- Do NOT use variables for `apiVersion` values — hardcode them directly on the resource
- Remove all unused variables
- Avoid concatenating variable names for conditional logic — use ternary expressions instead

## Resource Property Order

Within each resource, use this consistent property order:

```bicep
@description('...')
resource symbolicName 'Microsoft.Type/subtype@apiVersion' = {
  parent:           // if child resource
  scope:            // if cross-scope
  name:
  location:
  zones:
  sku:
  kind:
  scale:
  plan:
  identity:
  dependsOn:
  tags:
  properties:
}
```

## Symbolic Names & Resource References

- Use short, logical symbolic names: `storage`, `vm`, `vnet`, `nic`, `pip`
- Reference resource properties via symbolic name: `storage.id`, `storage.properties.primaryEndpoints.blob`
- Use `existing` keyword to reference pre-existing resources:

```bicep
resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' existing = {
  name: virtualNetworkName
}

// Then reference it:
subnetId: vnet.properties.subnets[0].id
```

- Never hardcode resource IDs — use `resourceId()` or symbolic `.id`
- Never hardcode endpoints — use symbolic references or `environment()` function:

```bicep
// CORRECT
storageUri: diagStorageAccount.properties.primaryEndpoints.blob

// WRONG — never do this
storageUri: '${diagStorageAccountName}.blob.core.windows.net'
```

## Outputs

- Output useful post-deployment values: endpoints, IP addresses, FQDNs
- **Never** output secrets, passwords, or account keys
- Use symbolic references in outputs:

```bicep
output storageEndpoint string = storage.properties.primaryEndpoints.blob
output vmPublicIp string = pip.properties.ipAddress
```

## Converting ARM JSON to Bicep

Use the Bicep decompiler as a starting point:

```bash
bicep decompile azuredeploy.json --outfile main.bicep
```

After decompiling, always:
1. Rename parameters and variables to camelCase (F2 in VS Code)
2. Rename resource symbolic names to short logical names (remove `Name` suffix if added by decompiler)
3. Remove `_var`, `_param`, `_resource` prefixes if present
4. Replace `concat()` with string interpolation: `'prefix-${paramName}-suffix'`
5. Replace `[resourceId(...)]` with symbolic `.id` references where possible
6. Remove all empty or null properties (`{}`, `[]`, `""`)
7. Verify top-level element order: `targetScope` → `param` → `var` → `resource`/`module` → `output`

## Modules

For reusable components, extract into `modules/` subfolder:

```bicep
module storage 'modules/storage.bicep' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: storageAccountName
    location: location
  }
}
```

## VM Images & Disks

- Use only Azure Marketplace or platform images — no custom images
- Always use `version: 'latest'` for platform image references
- Use managed disks for OS and data disks (implicit, not explicit resources)

## Linting & Validation

Always run before committing:

```bash
# Lint for best practice violations
az bicep lint --file main.bicep

# Validate against a real resource group
az deployment group validate \
  --resource-group <rg> \
  --template-file main.bicep \
  --parameters azuredeploy.parameters.json

# Build to ARM JSON to verify output (do NOT commit the .json)
az bicep build --file main.bicep
```

Common lint warnings to fix:
- Missing `@description()` on parameters
- Use of `any()` type — avoid it; use proper types instead
- Unused parameters or variables
- Hardcoded locations — use `param location string = resourceGroup().location`
- Outputs that expose secrets
