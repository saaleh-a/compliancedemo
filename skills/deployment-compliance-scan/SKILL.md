---
name: deployment-compliance-scan
description: |
  Read-only scheduled compliance scan for Azure Container Apps. Checks caller identity, trusted image provenance, labels, tags, revisions, and bootstrap-only state without modifying resources.
tools:
  - QueryLogAnalyticsByWorkspaceId
  - GetAzCliHelp
  - RunAzCliReadCommands
---

Scheduled compliance scans are detection-only.

Do not update Container Apps, activate or deactivate revisions, change traffic, rerun workflows, or attempt remediation. Report violations and state the approved interactive remediation path instead.

For each Container App in scope:
1. Resolve the Activity Log workspace through the subscription diagnostic setting named `activity-to-law`.
2. Validate Activity Log ingestion for the selected time range, then classify successful Container App writes by caller identity.
3. Inspect the active revision, deployed image digest, ACR image metadata, tags, and revision history.
4. Classify public placeholder images with `initial` deployment metadata and no compliant ACR image or previous revision as `NON-COMPLIANT BOOTSTRAP`.
5. Treat valid labels as supporting evidence only. Report `COMPLIANT` only when the caller is approved and trusted image provenance is verified; report `INVESTIGATE` if provenance cannot be verified.

Use `compliance_detection.md` for the KQL templates, caller IDs, and detailed decision tree.
