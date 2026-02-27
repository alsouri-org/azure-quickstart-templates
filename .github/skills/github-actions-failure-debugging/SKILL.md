---
name: github-actions-failure-debugging
description: Guide for debugging failing GitHub Actions workflows. Use this when asked to debug failing GitHub Actions workflows or when CI checks are failing on a pull request.
---

# GitHub Actions Failure Debugging Guide

## Process Overview

When CI is failing on a pull request, follow this structured process using the GitHub MCP server tools.

## Step 1 — Find the Failing Workflow Run

Use the `list_workflow_runs` tool to find recent runs for the pull request and identify which jobs failed:

```
list_workflow_runs(owner, repo, branch=<PR branch>)
```

Note the `run_id` of the failing run and which jobs have a `failure` or `cancelled` conclusion.

## Step 2 — Get an AI Summary of the Failure

Before reading raw logs (which can be very large), use the `summarize_job_log_failures` tool to get a concise summary:

```
summarize_job_log_failures(owner, repo, run_id=<run_id>)
```

This will identify what went wrong without filling context with thousands of log lines.

## Step 3 — Get Full Logs if Needed

If the summary is insufficient, retrieve detailed logs for specific jobs:

```
get_job_logs(owner, repo, job_id=<job_id>)
```

Or for the entire workflow run:

```
get_workflow_run_logs(owner, repo, run_id=<run_id>)
```

## Step 4 — Reproduce the Failure Locally

Once the cause is identified, reproduce it locally before fixing:

```bash
# Bicep lint errors
az bicep lint --file main.bicep

# Template validation errors
az deployment group validate \
  --resource-group <rg> \
  --template-file main.bicep \
  --parameters azuredeploy.parameters.json

# Build Bicep to check for compilation errors
az bicep build --file main.bicep

# Check for missing required files
ls azuredeploy.parameters.json metadata.json README.md
```

## Step 5 — Fix and Verify

Fix the issue, then verify locally before committing:
- Re-run the lint/validate commands above
- Confirm the fix addresses the exact error from the logs
- Commit only when the local check passes

---

## Common Failure Patterns in This Repo

### Bicep Lint / Compilation Errors
**Symptom:** `BCP` error codes in logs (e.g. `BCP018`, `BCP057`)

**Common causes:**
- Missing `@description()` decorator on a parameter
- Using `any()` type — replace with proper typed expression
- Referencing a non-existent resource property
- Incorrect module path

**Fix:** Run `az bicep lint --file main.bicep` locally and address all warnings/errors.

---

### ARM-TTK Test Failures
**Symptom:** `Test-AzTemplate` failures with messages like "Parameter Must Reference KeyVault" or "Location Should Not Be Hardcoded"

**Common causes:**
- Hardcoded location string (e.g. `"location": "eastus"`) — must use `[parameters('location')]`
- Missing `description` metadata on a parameter
- `apiVersion` stored in a variable instead of hardcoded on the resource
- `allowedValues` on the `location` parameter
- Secure parameter has a `defaultValue`

**Fix:** Read the specific test name from the logs, find the resource/parameter causing it, and apply the pattern from the `arm-template-authoring` or `bicep-authoring` skill.

---

### Missing Required Files
**Symptom:** CI fails with "Required file not found" or similar

**Required files every template folder must have:**
- `main.bicep` OR `azuredeploy.json` (not both)
- `azuredeploy.parameters.json`
- `metadata.json`
- `README.md`

**Fix:** Check which file is missing and create it with the correct content.

---

### Invalid metadata.json
**Symptom:** Schema validation error in CI logs

**Common causes:**
- `itemDisplayName` exceeds 60 characters
- `summary` exceeds 200 characters
- `description` exceeds 1000 characters
- `dateUpdated` is in wrong format (must be `YYYY-MM-DD`)
- Missing required fields

**Fix:** Validate against the schema:
```bash
# Check the metadata schema
cat test/metadata.schema.json

# Verify your metadata.json fields and lengths
cat metadata.json
```

---

### Template Deployment Failure
**Symptom:** CI deploys the template and the deployment fails

**Common causes:**
- Parameter default values are invalid in some regions or clouds
- Resource SKU not available in the test region
- Missing `dependsOn` causing a race condition
- Hardcoded endpoint or URI that doesn't work in USGov cloud

**Fix:**
1. Read the deployment error from the logs (ARM error code + message)
2. Reproduce with: `az deployment group validate --resource-group <rg> --template-file main.bicep --parameters azuredeploy.parameters.json`
3. For cross-cloud issues: avoid hardcoded endpoints, use `environment()` function or `reference()` for dynamic endpoints

---

### Parameters File Placeholders Missing
**Symptom:** CI fails with "Invalid parameter value" or deployment error on unique resources

**Common causes:**
- Using real values in `azuredeploy.parameters.json` instead of `GEN-*` placeholders
- Using `changeme` or empty strings as placeholder values

**Fix:** Replace with correct CI placeholders:
- Globally unique names → `GEN-UNIQUE` or `GEN-UNIQUE-N`
- Passwords → `GEN-PASSWORD`
- SSH keys → `GEN-SSH-PUB-KEY`
- GUIDs → `GEN-GUID`

---

### Multiple Templates Updated in One PR
**Symptom:** CI blocks the PR with "Multiple samples updated"

**Rule:** One PR must only update files within a single template folder.

**Fix:** Split the changes into separate PRs, one per template folder.
