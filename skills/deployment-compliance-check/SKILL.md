---
name: deployment-compliance-check
description: |
  Checks whether Azure Container App deployments comply with the organization's CI/CD-only deployment policy. Uses Activity Log caller identity, trusted image provenance, Docker image labels, and resource tags.
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
Non-compliant deployments must be flagged and reported. A revert is allowed only for an interactive investigation after the user explicitly approves it.
Scheduled-task executions are detection-only: they must not update a Container App, change revision activation or traffic, rerun a workflow, or attempt remediation even if an older task prompt requests it.
This policy ensures every production change is traceable to a code commit, reviewed via PR, and auditable through the pipeline.

How the Pipeline Works
GitHub Actions builds the Docker image with immutable compliance labels, pushes to ACR, which fires an Event Grid event. An Automation Runbook (running under a managed identity) picks up the event and updates the Container App via ARM. The key point: GitHub never authenticates to Azure AD directly — all Azure-side auth happens through managed identities inside Azure.

Data Sources
Activity Logs in Log Analytics
Activity Logs flow to the Log Analytics workspace via diagnostic settings. Use QueryLogAnalyticsByWorkspaceId to run KQL against the AzureActivity table.

To discover the authoritative workspace:

1. Resolve the subscription setting named `activity-to-law`:
   `az monitor diagnostic-settings subscription list --subscription <subscription-id> --query "value[?name=='activity-to-law'].workspaceId | [0]" -o tsv`
2. Resolve its customer ID:
   `az monitor log-analytics workspace show --ids <workspace-resource-id> --query customerId -o tsv`

Container App Resource Tags
Use RunAzCliReadCommands to check tags on the Container App.

Docker Image Labels and Provenance in ACR
The CI/CD pipeline bakes labels into every image at build time (deployed-by, commit-sha, pipeline-run-id, branch, repository, workflow). Labels are immutable within an image manifest after push, but are not authenticated provenance: any principal that can push to ACR can create the same labels. Treat labels as supporting evidence only; require trusted ACR push controls and, where available, a verified signed provenance attestation bound to the deployed image digest.

How to Detect Compliance
See compliance_detection.md for the detailed decision tree and well-known app IDs.

Step 0: Detect bootstrap-only state
Before proposing a revert, inspect the active revision, its image, tags, ACR repository inventory, and revision history. If the app still runs a public placeholder image, has `commit-sha=initial` or `pipeline-run-id=initial`, and has neither a compliant ACR image nor a prior revision, classify it as `NON-COMPLIANT BOOTSTRAP`. Do not attempt a rollback because no known-good target exists.

Step 1: Query Activity Logs
First run an unfiltered `AzureActivity` count and latest-timestamp query for the selected window to verify ingestion. Then query successful Container App write operations and extract `claims.appid` with `parse_json(tostring(Claims))`. See compliance_detection.md for the KQL template.

Step 2: Classify each deployment by caller
Well-known Azure Portal / CLI / PowerShell app IDs → NON-COMPLIANT
Caller contains @ (user principal) → NON-COMPLIANT
Known pipeline managed identity → proceed to Step 3
Unknown service principal → INVESTIGATE
If no retained write event is available, report caller identity as unavailable rather than inferring compliance from tags.
Caller identity ALWAYS takes precedence over tags.

Step 3: Verify image labels and trusted provenance
Even if the caller is the pipeline's managed identity, inspect the deployed image. Expected labels can identify an image that is clearly non-compliant when absent, but labels alone cannot prove that GitHub Actions built it because an ACR publisher can set arbitrary labels.

To classify the image:

Get the currently running image digest and retrieve its config from ACR.
Verify expected labels (deployed-by=pipeline, commit-sha, pipeline-run-id, branch, repository, workflow).
Verify that only the trusted CI identity can publish production images to the repository, and verify a signed provenance attestation if the registry supports it.
If labels are missing, the image is outside the configured ACR, or provenance is invalid → NON-COMPLIANT.
If labels are present but trusted publishing controls or signed provenance cannot be verified → INVESTIGATE.
Only report COMPLIANT when the caller is approved and the image has valid labels plus verified trusted provenance.

Step 4: Verify resource tags (secondary)
Compliant pipelines stamp tags like deployed-by=pipeline, pipeline-run-id, commit-sha, repository. Missing deployed-by tag is additional non-compliance evidence — but tags alone are weak because the Automation Runbook stamps them on every deploy, including ones triggered by manual ACR pushes.

Step 5: Generate compliance report
Report should include scan timestamp, time range, total/compliant/non-compliant counts, image label check results, and details of any violations.

Revert Procedures
IMPORTANT: Always get explicit user approval before any revert action. Scheduled-task executions must report the violation and stop; they cannot obtain or infer approval.

Option A — Reactivate previous Container App revision: list revisions, activate the last known-good one, shift traffic, deactivate the non-compliant revision.

Option B — Re-run the CI/CD pipeline to redeploy the last known compliant image from the approved pipeline.

Notes
Activity Logs may take 5-15 minutes to appear in Log Analytics
claims.appid values for Portal/CLI/PowerShell are well-known Microsoft constants (see compliance_detection.md)
Caller identity is authoritative; tags can be stale from previous deploys
Docker image labels are immutable supporting metadata, but trusted publisher controls or signed provenance provide the decisive image-build evidence
A manual ACR push can carry copied labels, so label-only checks cannot close the portal-push bypass
Never revert without user approval