# Compliance Detection Decision Tree

## Decision Flow

For each Container App deployment event (Microsoft.App/containerApps/write):

### 1. Classify the caller

Extract `claims.appid` after normalizing `Claims` to a string (in KQL: `extend ClaimsObj = parse_json(tostring(Claims)) | extend AppId = tostring(ClaimsObj["appid"])`).

- appid `c44b4083-3bb0-49c1-b47d-974e53cbdf3c` → Azure Portal → **NON-COMPLIANT**
- appid `04b07795-a710-4e84-bea4-c697bab44963` or `04b07795-8ddb-461a-bbee-02f9e1bf7b46` → Azure CLI → **NON-COMPLIANT**
- appid `1950a258-227b-4e31-a9cf-717495945fc2` → Azure PowerShell → **NON-COMPLIANT**
- appid `872cd9fa-d31f-45e0-9eab-6e460a02d1f1` → Visual Studio → **NON-COMPLIANT**
- appid `0a7bdc5c-7b57-40be-9939-d4c5fc7cd417` → Azure Mobile App → **NON-COMPLIANT**
- Caller contains `@` → User principal → **NON-COMPLIANT**
- Known pipeline managed identity → **go to step 2**
- Unknown service principal → **go to step 2**

For a scheduled scan, stop after classification and reporting. Do not attempt a revert, workflow rerun, revision operation, or any other resource modification.

### 1a. Handle bootstrap-only state

If no retained write event is available, inspect the active revision, its image, app tags, ACR repository inventory, and revision history. Classify the app as **NON-COMPLIANT BOOTSTRAP** when it runs a public placeholder image, has `commit-sha=initial` or `pipeline-run-id=initial`, and has neither a compliant ACR image nor a prior revision. Report the deployment-path gap; there is no known-good target to reactivate.

### 2. Verify image labels and trusted provenance

Even if the caller is the pipeline's managed identity, inspect the deployed image by immutable digest. Expected labels are necessary evidence, but not authenticated provenance: a principal that can push to ACR can create identical labels. Verify these labels and the trusted-publisher control:

- `deployed-by` = `pipeline`
- `commit-sha` = valid 40-char hex SHA
- `pipeline-run-id` = numeric GitHub Actions run ID
- `branch` = should be `main`
- `repository` = should match the expected repo
- `workflow` = should match the expected workflow name
- production ACR push access is restricted to the trusted CI identity, with a signed provenance attestation verified where available

**Valid labels + approved caller + verified trusted publisher/provenance** → **COMPLIANT**
**Valid labels + unknown caller or unverifiable trusted publisher/provenance** → **INVESTIGATE**
**Any labels missing or provenance invalid** → **NON-COMPLIANT**
**Image is outside the configured ACR or cannot be label-verified** → **NON-COMPLIANT**, unless it meets the bootstrap-only condition above

This catches the portal-push bypass: a manually published image can carry copied labels, so label checks must be paired with publisher authorization or signed provenance.

### 2a. Verify ACR publisher controls (read-only)

Use the configured registry name and resource ID to inspect inventory, registry posture, and all effective publishing permissions:

```bash
az acr repository list --name <acr-name> --subscription <subscription-id> -o json
az acr show --name <acr-name> --resource-group <resource-group> --subscription <subscription-id> --query "{adminUserEnabled:adminUserEnabled,publicNetworkAccess:publicNetworkAccess,policies:policies}" -o json
az role assignment list --scope <acr-resource-id> --include-inherited --subscription <subscription-id> --query "[].{principalId:principalId,principalType:principalType,role:roleDefinitionName,scope:scope}" -o json
```

`adminUserEnabled=true`, broad inherited `Owner` or `AcrPush` assignments, or an unavailable signed attestation means trusted publisher/provenance is **not verified**. These are posture signals, not proof that a specific image is compliant. If the deployed image is outside the configured ACR or the registry has no candidate image, classify it as `NON-COMPLIANT` (or `NON-COMPLIANT BOOTSTRAP` when the bootstrap conditions apply).

### 3. Check resource tags (secondary confirmation)

Look for `deployed-by=pipeline` and other pipeline tags on the Container App. These are the weakest signal because the Automation Runbook stamps them on every deploy regardless of how the image got into ACR. Tags alone cannot distinguish a legitimate pipeline deploy from a portal-push-via-Event-Grid deploy.

**Caller identity always takes precedence over tags. Image labels always take precedence over tags.**

## Well-Known Azure Application IDs

| Application ID | Name |
|---|---|
| c44b4083-3bb0-49c1-b47d-974e53cbdf3c | Azure Portal |
| 04b07795-a710-4e84-bea4-c697bab44963 | Microsoft Azure CLI |
| 04b07795-8ddb-461a-bbee-02f9e1bf7b46 | Microsoft Azure CLI |
| 1950a258-227b-4e31-a9cf-717495945fc2 | Microsoft Azure PowerShell |
| 872cd9fa-d31f-45e0-9eab-6e460a02d1f1 | Visual Studio |
| 0a7bdc5c-7b57-40be-9939-d4c5fc7cd417 | Microsoft Azure Mobile App |

## KQL Template

First validate that the selected workspace is ingesting activity for the scan window:

```kql
AzureActivity
| where TimeGenerated > ago(##timeRange##)
| summarize totalRows=count(), latestRow=max(TimeGenerated)
```

Then run the scoped deployment query:

```kql
AzureActivity
| where TimeGenerated > ago(##timeRange##)
| where OperationNameValue =~ "Microsoft.App/containerApps/write"
| where ActivityStatusValue =~ "Succeeded"
| where ResourceGroup =~ "##resourceGroup##"
| extend ClaimsObj = parse_json(tostring(Claims))
| extend AppId = tostring(ClaimsObj["appid"])
| extend CallerType = case(
    AppId == "c44b4083-3bb0-49c1-b47d-974e53cbdf3c", "AzurePortal",
    AppId in ("04b07795-a710-4e84-bea4-c697bab44963", "04b07795-8ddb-461a-bbee-02f9e1bf7b46"), "AzureCLI",
    AppId == "1950a258-227b-4e31-a9cf-717495945fc2", "AzurePowerShell",
    Caller contains "@", "UserPrincipal",
    "ServicePrincipal"
  )
| project TimeGenerated, Caller, CallerIpAddress, CallerType, AppId, Resource, CorrelationId
| order by TimeGenerated desc
```

Set `##timeRange##` and `##resourceGroup##` for the scan. An empty scoped result is not proof of compliance: state that caller evidence is unavailable for the retention window and combine it with current image, label, tag, and revision checks.

For the direct Activity Log fallback, query the exact Container App resource with `az monitor activity-log list --resource-id <container-app-resource-id> --offset 90d`, project `resourceId`, and verify every returned row exactly matches the target resource ID. Do not treat unrelated or uncorrelated rows as caller evidence. Do not use a fixed start time older than 90 days; if no matching deployment is retained, report caller evidence as unavailable rather than inferring compliance.

Note: A service-principal caller is only potentially compliant. Verify the image digest, labels, and trusted publisher or signed provenance before reporting a compliant deployment.

## Signal Priority

1. **Caller identity** — who made the ARM call (from Activity Log)
2. **Trusted image provenance** — publisher authorization or a signed attestation bound to the deployed digest
3. **Docker image labels** — immutable supporting metadata, not authenticated provenance
4. **Resource tags** — what does the Container App say (weakest, can be misleading)