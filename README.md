# Deployment Compliance Monitoring with SRE Agent

Detects and responds to non-compliant Azure Container App deployments using SRE Agent, Activity Logs, and KQL analysis.

## What it does

- **Compliant**: A deployment has an approved caller and verified trusted image provenance; labels such as `deployed-by=pipeline`, `commit-sha`, and `pipeline-run-id` are supporting evidence.
- **Non-compliant**: Deployments via Azure Portal or ad-hoc CLI, images without required metadata, images outside the configured registry, or invalid provenance.

When a Container App deployment is detected:
1. **Alert fires** → Activity Log alert on `Microsoft.App/containerApps/write`
2. **SRE Agent investigates** → Uses `deployment-compliance-check` for interactive investigation or `deployment-compliance-scan` for scheduled detection
3. **Classifies** → Portal or CLI caller = non-compliant; service-principal caller requires image provenance verification
4. **For non-compliant** → Scheduled scans report only; interactive investigations activate the approval hook before any revert
5. **For compliant** → Confirms and closes the alert

## Architecture

```
GitHub Actions (push to main)
    ↓
Build Docker image → Push to ACR
    ↓
az containerapp update (with compliance tags)
    ↓
Activity Log: containerApps/write
    ↓                          ↓
Alert Rule fires          Scheduled Task (every 30 min)
    ↓                          ↓
SRE Agent Response Plan   deployment-compliance-scan (read-only)
    ↓                          ↓
deployment-compliance-check skill  Report violation; do not mutate resources
    ↓
Compliant? ──yes──► Close alert
    ↓ no
Activate approval hook → Wait for user → Revert revision
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

# 2. Configure SRE Agent (connectors, skill, hook, response plan, scheduled task)
bash scripts/post-deploy.sh

# 3. Create service principal for GitHub Actions (run in Azure Portal Cloud Shell)
az ad sp create-for-rbac --name "compliancedemo-deploy" \
  --role Contributor \
  --scopes "/subscriptions/<SUB_ID>/resourceGroups/rg-compliancedemo" \
  --json-auth

# 4. Add GitHub secrets (see below)

# 5. Authorize GitHub connector
#    Open the OAuth URL printed by post-deploy.sh in your browser
```

### GitHub Secrets & Variables

| Type | Name | Value |
|------|------|-------|
| Secret | `ACR_USERNAME` | ACR admin username |
| Secret | `ACR_PASSWORD` | ACR admin password |
| Secret | `AZURE_CREDENTIALS` | JSON output from `az ad sp create-for-rbac --json-auth` |
| Variable | `ACR_NAME` | ACR name (without `.azurecr.io`) |

## Testing Compliance

```bash
# Compliant deployment — push a code change via PR/merge
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
