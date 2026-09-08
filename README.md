# Deployment Compliance Monitoring with SRE Agent

Detects and responds to non-compliant Azure Container App deployments using SRE Agent, Activity Logs, and KQL analysis.

## What it does

- **Compliant**: A deployment made by the approved CI/CD identity whose image has all immutable pipeline labels, including `deployed-by=pipeline`, `commit-sha`, and `pipeline-run-id`.
- **Non-compliant**: A deployment from Azure Portal, ad-hoc CLI, or PowerShell, or an image missing required immutable labels.

When a Container App deployment is detected:
1. **Alert fires** → Activity Log alert on `Microsoft.App/containerApps/write`
2. **SRE Agent investigates** → Runs the `deployment-compliance-check` skill against Activity Log, ACR, tags, and revision state
3. **Classifies** → Caller identity is primary evidence; immutable image labels are required for a compliant result
4. **For non-compliant** → Reports the violation and identifies a rollback only when a healthy, label-compliant target exists
5. **For compliant** → Confirms the evidence and closes the alert when applicable

Scheduled scans are detection-only. They never change revisions, traffic, workflows, or images. A separate interactive response may modify a deployment only after the approval hook permits it and the replacement target is verified.

## Architecture

```
GitHub Actions (push to main)
    ↓
Build Docker image with immutable labels → Push to ACR
    ↓
Verified deployment path: direct Container App update
or provisioned Event Grid → Automation/managed identity
    ↓
Activity Log: containerApps/write
    ↓                          ↓
Alert Rule fires          Scheduled Task (every 30 min, detection-only)
    ↓                          ↓
SRE Agent Response Plan   SRE Agent Compliance Scan
    ↓                          ↓
deployment-compliance-check deployment-compliance-scan
(interactive, approval-gated)  (read-only)
    ↓                          ↓
Report violation → verify       Report status and candidate targets only
healthy compliant target →
approval hook → interactive rollback
```

## Deployed Resources

| Resource | Purpose |
|----------|---------|
| Container App | Sample workload (Express.js API) |
| ACR | Container image registry |
| Log Analytics Workspace | Activity Log storage + KQL queries |
| Activity Log Alert | Triggers on Container App write operations |
| SRE Agent | AI agent with Kusto connector, skill, hook, scheduled task |

## Setup

### Prerequisites
- Azure CLI + Azure Developer CLI (`azd`)
- Azure subscription with permissions to create resources
- GitHub account

### Deploy

```bash
# 1. Provision infrastructure
azd init
azd provision

# 2. Configure the GitHub OIDC deployment identity and the repository variables
#    documented below. The workflow builds, pushes, and updates the Container
#    App directly; no Event Grid or Automation deployment path is required.

# 3. Configure SRE Agent (connectors, skills, hook, response plan, scheduled task)
#    Allowlist the application ID of the federated identity that performs
#    Microsoft.App/containerApps/write.
APPROVED_PIPELINE_CALLER_IDS="<github-oidc-client-id>" bash scripts/post-deploy.sh

# 4. Authorize GitHub connector
#    Open the OAuth URL printed by post-deploy.sh in your browser
```

### GitHub OIDC Deployment Configuration

Create an Entra application or user-assigned managed identity with a federated credential restricted to this repository's `main` branch. Grant that identity `AcrPush` on the provisioned ACR plus `Container Apps Contributor` and `Tag Contributor` on the target Container App. `Tag Contributor` is required because the workflow stamps deployment provenance tags after updating the image. The workflow does not use ACR admin credentials.

Set these repository-level GitHub Actions variables from the `azd provision` outputs and your Entra configuration:

| Name | Value |
|------|-------|
| `ACR_NAME` | `ACR_NAME` output, without `.azurecr.io` |
| `RESOURCE_GROUP_NAME` | `RESOURCE_GROUP_NAME` output |
| `CONTAINER_APP_NAME` | `CONTAINER_APP_NAME` output |
| `AZURE_CLIENT_ID` | Application or managed identity client ID |
| `AZURE_TENANT_ID` | Entra tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Target Azure subscription ID |

### Refreshing Agent Configuration

After changing either compliance skill, detection rules, approval hook, scheduled-task template, or the approved CI writer identity, rerun `bash scripts/post-deploy.sh` interactively. Set `APPROVED_PIPELINE_CALLER_IDS` to a comma-separated allowlist of application IDs for the identities that actually perform `Microsoft.App/containerApps/write`. Without a verified writer identity, leave the allowlist unset so scans fail closed as `INVESTIGATE`. The script uploads the approval-gated interactive skill and the read-only scheduled-scan skill, then recreates `compliance-scan` with the detection-only prompt. Do not run this script from a scheduled scan because it changes agent configuration and task state.

## Testing Compliance

```bash
# Compliant deployment — first provision a verified path that updates the
# Container App under the application ID configured in APPROVED_PIPELINE_CALLER_IDS.
# Create and merge a PR into main; the workflow only triggers on main pushes.
echo "// test" >> src/api/server.js
git add . && git commit -m "test deployment"
git push origin HEAD
# Create a pull request targeting main and merge it after review.

# Verify the direct CI/CD or provisioned Event Grid/Automation deployment completed,
# then ask the SRE Agent to inspect the Container App write and image labels.
"Check deployment compliance for the last hour"

# Non-compliant deployment — change via Portal
# Go to Azure Portal → Container App → Edit and Deploy → Change something
```

## Files

```
├── .github/workflows/deploy-container-app.yml  # CI/CD pipeline
├── infra/                                       # Bicep infrastructure
├── scripts/post-deploy.sh                       # SRE Agent configuration
├── skills/deployment-compliance-check/          # Interactive, approval-gated skill
├── skills/deployment-compliance-scan/           # Read-only scheduled scan skill
├── hooks/deployment-compliance-approval.yaml    # Approval hook for reverts
└── src/api/                                     # Sample Express.js app
```
