# Deployment Compliance Monitoring with SRE Agent

Detects and responds to non-compliant Azure Container App deployments using SRE Agent, Activity Logs, and KQL analysis.

## What it does

- **Compliant**: Deployments via this CI/CD pipeline (GitHub Actions) — tagged with `deployed-by=pipeline`, `commit-sha`, `pipeline-run-id`
- **Non-compliant**: Deployments via Azure Portal or ad-hoc CLI — detected by `claims.appid` in Activity Log

When a Container App deployment is detected:
1. **Alert fires** → Activity Log alert on `Microsoft.App/containerApps/write`
2. **SRE Agent investigates** → Runs the `deployment-compliance-check` skill via KQL
3. **Classifies** → Portal app ID `c44b4083...` = non-compliant; CI/CD service principal = compliant
4. **For non-compliant** → Activates approval hook, recommends revert to previous revision
5. **For compliant** → Confirms and closes the alert

## Architecture

```
GitHub Actions (push to main)
    ↓
Build labeled image → Push to ACR
    ↓
Event Grid image-push event
    ↓
Automation Runbook managed identity → az containerapp update
    ↓
Activity Log: containerApps/write
    ↓                          ↓
Alert Rule fires          Scheduled Task (every 30 min)
    ↓                          ↓
SRE Agent Response Plan   SRE Agent Compliance Scan
    ↓
deployment-compliance-check skill (KQL queries)
    ↓
Compliant? ──yes──► Close alert
    ↓ no
Activate approval hook → Wait for user → Revert revision
```

> [!IMPORTANT]
> `azd provision` creates only the bootstrap Container App and ACR. It does not provision the Event Grid subscription or Automation Runbook shown above. Before dispatching the workflow, implement that deployment path and set the workflow's `ACR_NAME` to the registry emitted by the current `azd` environment. The checked-in workflow otherwise pushes only to its hard-coded registry and cannot update the provisioned Container App.

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

# 2. Configure SRE Agent (connectors, skill, hook, response plan, scheduled task)
bash scripts/post-deploy.sh

# 3. Provision and validate the Event Grid → Automation Runbook deployment path.
#    This is not created by the Bicep templates today.

# 4. Set .github/workflows/deploy-container-app.yml ACR_NAME to the ACR emitted by azd.
#    Add the ACR credentials required by that workflow (see below).

# 5. Authorize the GitHub connector
#    Open the OAuth URL printed by post-deploy.sh in your browser
```

### GitHub Secrets

| Secret | Value |
|------|-------|
| `ACR_USERNAME` | Username for the ACR configured in the workflow |
| `ACR_PASSWORD` | Password for the ACR configured in the workflow |

The checked-in workflow does not use `AZURE_CREDENTIALS`. Do not create a Contributor service principal merely for this sample unless the workflow is deliberately changed to make direct Azure deployment calls.

## Testing Compliance

```bash
# After the Event Grid → Automation Runbook path is provisioned and validated:
# push a code change via PR/merge to trigger a compliant deployment.
echo "// test" >> src/api/server.js
git add . && git commit -m "test deployment" && git push

# Non-compliant deployment — change via Portal
# Go to Azure Portal → Container App → Edit and Deploy → Change something

# Ask SRE Agent
"Check deployment compliance for the last hour"
```

## Files

```
├── .github/workflows/deploy-container-app.yml  # CI/CD pipeline
├── infra/                                       # Bicep infrastructure
├── scripts/post-deploy.sh                       # SRE Agent configuration
├── skills/deployment-compliance-check/          # KQL-based compliance skill
├── hooks/deployment-compliance-approval.yaml    # Approval hook for reverts
└── src/api/                                     # Sample Express.js app
```
