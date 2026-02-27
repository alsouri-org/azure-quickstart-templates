---
name: arm-template-authoring
description: Guide for authoring ARM JSON templates (azuredeploy.json) in this repo. Use this when creating or editing azuredeploy.json or any ARM JSON template files, or when asked about ARM template syntax, functions, apiVersions, or validation.
---

# ARM JSON Template Authoring Guide for Azure Quickstart Templates

## File Naming

- The entry point file must be named `azuredeploy.json`
- If `main.bicep` exists in the same folder, do **NOT** also create `azuredeploy.json` — it is auto-generated on merge
- Parameters file is always `azuredeploy.parameters.json`
- Nested templates go in the `nestedtemplates/` subfolder

## Top-Level Structure & Property Order

Always use this exact order:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "variables": {},
  "resources": [],
  "outputs": {}
}
```

Omit `functions` and `apiProfile` unless explicitly required. Do not include empty top-level sections except `parameters`, `variables`, `resources`, and `outputs`.

## Parameters

Every parameter must have a `metadata.description`:

```json
"parameters": {
  "location": {
    "type": "string",
    "defaultValue": "[resourceGroup().location]",
    "metadata": {
      "description": "Location for all resources."
    }
  },
  "storageAccountName": {
    "type": "string",
    "defaultValue": "[concat('storage', uniqueString(resourceGroup().id))]",
    "metadata": {
      "description": "Name of the storage account."
    }
  },
  "adminUsername": {
    "type": "string",
    "metadata": {
      "description": "Admin username for the VM."
    }
  },
  "adminPassword": {
    "type": "securestring",
    "metadata": {
      "description": "Admin password for the VM."
    }
  },
  "storageAccountSku": {
    "type": "string",
    "defaultValue": "Standard_LRS",
    "allowedValues": [
      "Standard_LRS",
      "Standard_GRS",
      "Standard_ZRS",
      "Premium_LRS"
    ],
    "metadata": {
      "description": "The SKU of the storage account."
    }
  },
  "vmCount": {
    "type": "int",
    "defaultValue": 2,
    "minValue": 2,
    "maxValue": 8,
    "metadata": {
      "description": "Number of VMs to deploy."
    }
  }
}
```

Rules for parameters:
- `location` must default to `[resourceGroup().location]` — never add `allowedValues` to it
- Credentials (`securestring` type) must never have a `defaultValue`
- Use `uniqueString(resourceGroup().id)` in defaults for globally unique names
- Use `allowedValues` only for exclusive/constrained lists, not inclusive ones (e.g. not for all VM SKUs)
- Use `minValue`/`maxValue` for numeric ranges

## Variables

- Use variables for values used multiple times or complex expressions
- Do NOT use variables for `apiVersion` values — hardcode them on each resource
- Remove all unused variables
- Avoid string concatenation for conditional logic — use `if()` function instead:

```json
"variables": {
  "storageAccountName": "[concat('storage', uniqueString(resourceGroup().id))]",
  "subnetRef": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('virtualNetworkName'), parameters('subnetName'))]"
}
```

## Resources — Property Order

Within each resource, use this consistent order:

```json
{
  "comments": "optional",
  "condition": true,
  "type": "Microsoft.Compute/virtualMachines",
  "apiVersion": "2023-03-01",
  "name": "[parameters('vmName')]",
  "location": "[parameters('location')]",
  "zones": [],
  "sku": {},
  "kind": "",
  "plan": {},
  "identity": {},
  "copy": {},
  "dependsOn": [],
  "tags": {},
  "properties": {}
}
```

- Always use `[parameters('location')]` for `location` — never hardcode a region string
- Omit empty properties (`{}`, `[]`, `""`) unless required by the schema
- Place `comments` only when genuinely clarifying — keep them brief

## apiVersions

- Hardcode `apiVersion` directly on each resource — do NOT put them in variables
- Use stable API versions, not preview (avoid `-preview` suffix in production templates)
- Select the latest stable version available for the resource type

## dependsOn

Only use when necessary — implicit dependencies from `reference()` and `resourceId()` are handled automatically:

```json
"dependsOn": [
  "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
]
```

- Use the resource name string or `resourceId()` call as the dependency value
- Do NOT use full `resourceId()` if the name alone is unambiguous within the template
- Conditional resources are automatically removed from the dependency graph when `condition` is false

## resourceId() and reference()

Always use `resourceId()` to construct resource IDs — never hardcode them:

```json
"id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('vnetName'), parameters('subnetName'))]"
```

Use `reference()` to access runtime properties (endpoints, IPs, keys):

```json
"storageUri": "[reference(variables('diagStorageAccountName'), '2021-02-01').primaryEndpoints.blob]"
```

For resources in a different resource group:

```json
"storageUri": "[reference(resourceId(parameters('storageRg'), 'Microsoft.Storage/storageAccounts', variables('storageName')), '2021-02-01').primaryEndpoints.blob]"
```

**Never hardcode endpoints** — always use `reference()` or `environment()`:

```json
// WRONG
"hostName": "[concat(parameters('webAppName'), '.azurewebsites.net')]"

// CORRECT - use reference() to get the actual hostname
"hostName": "[reference(resourceId('Microsoft.Web/sites', parameters('webAppName'))).defaultHostName]"
```

## Deployment Artifacts (Nested Templates & Scripts)

If the template uses nested templates or scripts, add these standard parameters:

```json
"_artifactsLocation": {
  "type": "string",
  "defaultValue": "[deployment().properties.templateLink.uri]",
  "metadata": {
    "description": "The base URI where artifacts required by this template are located including a trailing '/'"
  }
},
"_artifactsLocationSasToken": {
  "type": "securestring",
  "defaultValue": "",
  "metadata": {
    "description": "The sasToken required to access _artifactsLocation."
  }
}
```

Construct artifact URIs using `uri()`:

```json
"variables": {
  "nestedTemplateUri": "[uri(parameters('_artifactsLocation'), concat('nestedtemplates/vm.json', parameters('_artifactsLocationSasToken')))]",
  "scriptUri": "[uri(parameters('_artifactsLocation'), concat('scripts/configure.sh', parameters('_artifactsLocationSasToken')))]"
}
```

Never use raw `raw.githubusercontent.com` URLs — always use `_artifactsLocation`.

## VM Images & Disks

- Use only Azure Marketplace or platform images — no custom images
- Use `"version": "latest"` for platform image references
- Use managed disks (implicit) for OS and data disks:

```json
"osDisk": {
  "caching": "ReadWrite",
  "createOption": "FromImage"
},
"dataDisks": [
  {
    "diskSizeGB": "[parameters('dataDiskSizeGB')]",
    "lun": 0,
    "createOption": "Empty"
  }
]
```

## Outputs

- Output useful post-deployment values: endpoints, IP addresses, FQDNs, connection strings
- **Never** output secrets, passwords, or account keys

```json
"outputs": {
  "storageEndpoint": {
    "type": "string",
    "value": "[reference(variables('storageAccountName')).primaryEndpoints.blob]"
  },
  "vmPublicIpAddress": {
    "type": "string",
    "value": "[reference(resourceId('Microsoft.Network/publicIPAddresses', variables('pipName'))).ipAddress]"
  }
}
```

## Validation Before Committing

```bash
# Validate the template
az deployment group validate \
  --resource-group <rg> \
  --template-file azuredeploy.json \
  --parameters azuredeploy.parameters.json

# Run arm-ttk tests (if available locally)
Import-Module arm-ttk
Test-AzTemplate -TemplatePath .
```
