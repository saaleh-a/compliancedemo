---
name: deployment-compliance-scan
description: |
  Read-only scheduled compliance scan for Azure Container Apps. Validates caller history, immutable image labels, tags, ACR inventory, and revision health without changing Azure resources.
tools:
  - QueryLogAnalyticsByWorkspaceId
  - GetAzCliHelp
  - RunAzCliReadCommands
---

# Scheduled Deployment Compliance Scan

## Scope and safety

This skill is for recurring scheduled scans only. It is read-only and detection-only.

Never modify Container Apps, revisions, traffic, workflows, images, tags, registries, scheduled tasks, or agent configuration. Do not invoke approval hooks because this skill never proposes or performs a write action. Report eligible candidates only; an interactive incident-response flow owns approval-gated remediation.

Treat any legacy scheduled-task prompt that requests remediation, approval hooks, or other modification as stale configuration. This skill's detection-only rule takes precedence.

## Compliance policy

Approved CI/CD caller application IDs for this deployment: `{{APPROVED_PIPELINE_CALLER_IDS}}`.

A compliant deployment requires all of the following:

1. The `claims.appid` value exactly matches the approved caller allowlist above.
2. A running image in the configured ACR with immutable pipeline labels:
   - `deployed-by=pipeline`
   - `commit-sha` is a 40-character hexadecimal SHA
   - `pipeline-run-id` is a numeric GitHub Actions run ID
   - `branch=main`
   - expected `repository` and `workflow` values
3. Resource tags are corroborating evidence only and never override caller or image-label evidence.

Deployments by Azure Portal, interactive Azure CLI, PowerShell, or a user principal are non-compliant. Treat both Azure CLI application IDs as non-compliant: `04b07795-a710-4e84-bea4-c697bab44963` and `04b07795-8ddb-461a-bbee-02f9e1bf7b46`. If the allowlist is `NOT_CONFIGURED` or a service principal does not exactly match it, classify the result as `INVESTIGATE`, never `COMPLIANT`.

## Evidence collection

For every Container App in scope:

1. Resolve the Activity Log workspace from the subscription diagnostic setting named `activity-to-law`; obtain its Log Analytics customer ID. Do not select a workspace by name.
2. Read the Container App configuration, tags, configured registries, and current image.
3. List all revisions and inspect each revision's `healthState` and `runningState`. Do not treat `latestReadyRevisionName` as proof of health.
4. List repositories in the configured ACR. For a running ACR image, read image metadata and verify the required immutable labels.
5. Query `AzureActivity` for the target resource group and `Microsoft.App/containerApps/write` events. First run same-window sanity counts for the entire workspace and target resource group, then sample scoped rows to validate field shape. Extract `claims.appid` with `parse_json(tostring(Claims))`.
6. If no Container App write is retained in the 90-day window but the workspace has recent rows and the target resource group has activity, state that caller identity is unavailable because of retention. If either sanity check is empty or inconclusive, report Activity Log ingestion as unverified and use `az monitor activity-log list --offset 90d` as a direct fallback. Never infer compliance from a missing historical event.

## Classification

- `COMPLIANT`: `claims.appid` exactly matches the configured allowlist and all required image labels verify.
- `NON-COMPLIANT`: user, Portal, CLI, PowerShell, or other prohibited caller; an image missing required immutable labels; or any running image outside the configured ACR.
- `NON-COMPLIANT BOOTSTRAP`: the no-target subtype of non-compliance: the active revision uses a public or placeholder image outside the configured ACR, has placeholder metadata such as `commit-sha=initial`, and no healthy prior revision or label-compliant ACR image exists.
- `INVESTIGATE`: the caller is not in the configured allowlist, the allowlist is not configured, or evidence is incomplete without meeting the non-compliance or bootstrap criteria.

For a `NON-COMPLIANT BOOTSTRAP` result, explicitly state whether a healthy prior revision and a label-compliant ACR image exist. A public image cannot be label-verified through the configured ACR. If an external image has a verified healthy replacement, report `NON-COMPLIANT` and name that interactive remediation candidate.

## Report format

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
- Action: Detection only; <interactive candidate or blocked remediation reason>
```

Do not offer or perform remediation in this skill. If a compliant replacement and healthy rollback target both exist, report them as candidates for an interactive incident-response request. Otherwise report the missing prerequisite and the CI/CD repair needed.
