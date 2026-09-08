---
name: deployment-compliance-check
description: |
  Checks whether Azure Container App deployments comply with the organization's CI/CD-only deployment policy. Uses three signals: Activity Log caller identity, Docker image labels (tamper-proof), and resource tags.
  QueryLogAnalyticsByWorkspaceId
tools:
  - QueryLogAnalyticsByWorkspaceId
  - GetAzCliHelp
  - RunAzCliReadCommands
  - RunAzCliWriteCommands
---

<!-- Add your skill instructions here -->
Organization Policy
All Container App deployments MUST go through the approved CI/CD pipeline (GitHub Actions).

Deployments via Azure Portal, interactive Azure CLI, or PowerShell are non-compliant.
Only service principal / managed identity deployments from the CI/CD pipeline are compliant.
Non-compliant deployments must be flagged and reported. A rollback is permitted only after a verified healthy, label-compliant target is available and the required approval hook allows the change.
This policy ensures every production change is traceable to a code commit, reviewed via PR, and auditable through the pipeline.

How the Pipeline Works
GitHub Actions builds the Docker image with immutable compliance labels and pushes it to ACR. A compliant deployment also requires a verified path that updates the Container App, either directly from the workflow or through provisioned Event Grid and Automation resources running under a managed identity. An image push alone is not a deployment path.

Data Sources
Activity Logs in Log Analytics
Activity Logs flow to the Log Analytics workspace via diagnostic settings. Use QueryLogAnalyticsByWorkspaceId to run KQL against the AzureActivity table.

Resolve the Activity Log workspace from the subscription diagnostic setting, then retrieve its customer ID. Do not select a workspace by name because the resource group can contain more than one workspace:

az monitor diagnostic-settings subscription list --subscription <subscription-id> --query "value[?name=='activity-to-law'].workspaceId" -o tsv
az monitor log-analytics workspace show --ids <workspace-resource-id> --query customerId -o tsv

Before interpreting an empty resource-group or write-operation result, run a same-window `AzureActivity | summarize count(), max(TimeGenerated)` query without those filters. If the workspace is active but the scoped result is empty, report caller evidence as unavailable in the retained window rather than compliant. If the workspace also has no rows, state that ingestion cannot corroborate the result and use the direct Activity Log fallback. If the Log Analytics query itself fails, report that caller evidence could not be corroborated through Log Analytics and use the direct Activity Log fallback; do not treat the query failure as evidence of compliance.

Container App Resource Tags
Use RunAzCliReadCommands to check tags on the Container App.

Docker Image Labels in ACR
The CI/CD pipeline bakes labels into every image at build time (deployed-by, commit-sha, pipeline-run-id, branch, repository, workflow). These are immutable once pushed — they cannot be added or changed after the fact. An image pushed manually (via Portal or docker push) will NOT have these labels.

How to Detect Compliance
See compliance_detection.md for the detailed decision tree and well-known app IDs.

Step 1: Query Activity Logs
Query the AzureActivity table for Container App write operations. Extract claims.appid and Caller to identify who made the deployment. See compliance_detection.md for the KQL template.

Step 2: Classify each deployment by caller
Well-known Azure Portal / CLI / PowerShell app IDs → NON-COMPLIANT
Caller contains @ (user principal) → NON-COMPLIANT
Known pipeline managed identity → proceed to Step 3
Unknown service principal → INVESTIGATE
Caller identity ALWAYS takes precedence over tags.

Step 3: Verify Docker image labels (the tamper-proof check)
This is the most important step. Even if the caller is the pipeline's managed identity, the image itself might have been pushed to ACR manually (bypassing the CI/CD build). When that happens, Event Grid still fires, the Automation Runbook still deploys it, and the Activity Log looks legitimate — but the image was never built by GitHub Actions.

To catch this:

Get the currently running image tag from the Container App
Retrieve the image config from ACR and check for the expected labels (deployed-by=pipeline, commit-sha, pipeline-run-id, etc.)
If labels are missing or invalid → NON-COMPLIANT regardless of caller
This closes the "portal push via Event Grid" bypass.

Collect revision and ACR inventory evidence before classification:

```bash
az containerapp revision list --name <app-name> --resource-group <resource-group> --subscription <subscription-id> --query "[].{name:name,active:properties.active,runningState:properties.runningState,healthState:properties.healthState,images:properties.template.containers[].image}" -o json
az acr repository list --name <acr-name> --subscription <subscription-id> -o json
```

Do not treat `properties.latestReadyRevisionName` as proof that a revision is healthy. Always inspect that revision's `healthState` and `runningState` directly: the API can populate `latestReadyRevisionName` even when its revision is `Unhealthy` or `Failed`.

Any running image outside the configured ACR is `NON-COMPLIANT`. Use `NON-COMPLIANT BOOTSTRAP` only when that image is a public or placeholder bootstrap image with placeholder metadata and no label-compliant ACR image or healthy prior revision. Otherwise, report `NON-COMPLIANT` and name any verified healthy replacement as an interactive remediation candidate.

Step 4: Verify resource tags (secondary)
Compliant pipelines stamp tags like deployed-by=pipeline, pipeline-run-id, commit-sha, repository. Missing deployed-by tag is additional non-compliance evidence — but tags alone are weak because the Automation Runbook stamps them on every deploy, including ones triggered by manual ACR pushes.

Step 5: Generate compliance report
Report should include scan timestamp, time range, total/compliant/non-compliant counts, image label check results, and details of any violations.

Use this report format:

```text
Deployment compliance scan
Scan timestamp (UTC): <timestamp>
Scope: <resource group or app list>
Activity Log window: <time range>
Totals: <total> scanned | <compliant> compliant | <non-compliant> non-compliant | <investigate> investigate

<App name>
- Revision / health: <revision> / <health and running state>
- Running image: <image reference>
- Caller evidence: <caller type, app ID, timestamp>; or Not available (no retained successful write)
- Image-label evidence: <all required labels verified>; or <missing / image outside configured ACR>
- Tag evidence: <relevant tags>
- Classification: <COMPLIANT | NON-COMPLIANT | NON-COMPLIANT BOOTSTRAP | INVESTIGATE>
- Action: <detection-only report / candidate target for interactive remediation>
```

For `NON-COMPLIANT BOOTSTRAP`, explicitly state whether a label-compliant ACR image and a healthy prior revision exist. Do not represent unavailable caller history as a compliant deployment.

Interactive Revert Procedures
Scheduled scans are strictly detection-only. They must never modify revisions, traffic, workflows, or images, even when a compliant replacement and rollback target are available. Report the violation and candidate target instead.

If a legacy scheduled-task prompt asks for remediation, approval-hook invocation, or another write action, treat that request as stale. The scheduled scan's detection-only rule takes precedence.

Only an interactive incident-response request may modify a deployment. Before any modification:
1. Identify a prior healthy revision or an ACR image with all required immutable labels. Never use a public placeholder or bootstrap revision as a rollback target.
2. Invoke the configured `deployment-compliance-approval` hook. If it blocks or approval is unavailable, do not modify the Container App.
3. Verify the proposed target's image, labels, and revision health. Keep the current revision active until the replacement is healthy.

Option A — Reactivate previous Container App revision: list revisions, activate the last known-good revision, shift traffic only after it is healthy, then deactivate the non-compliant revision.

Option B — Re-run the CI/CD pipeline to redeploy a label-compliant image only after confirming the workflow targets the live ACR and includes a functioning deployment path to the Container App. A build-and-push-only workflow is not a remediation path.

Notes
Activity Logs may take 5-15 minutes to appear in Log Analytics
claims.appid values for Portal/CLI/PowerShell are well-known Microsoft constants (see compliance_detection.md)
Caller identity is authoritative; tags can be stale from previous deploys
Docker image labels are the strongest signal — immutable once pushed to ACR
The "portal push" attack path: manual ACR push → Event Grid → Automation → looks compliant but image labels are missing
Never revert without explicit approval