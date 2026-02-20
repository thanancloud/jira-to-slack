# CloudBees Workflow Setup Guide

## Overview
This guide will help you set up the Jira Bug Aging Report in CloudBees CI with AWS Bedrock (Claude AI) using OIDC authentication.

## Prerequisites
1. CloudBees CI access
2. AWS account with Bedrock enabled
3. Jira API token
4. EngOps support for IAM role creation

---

## Step 1: Request IAM Role from EngOps

Contact EngOps team and reference ticket: **[OPS-20629](https://cloudbees.atlassian.net/browse/OPS-20629)**

**Request Details:**
```
Subject: OIDC IAM Role for Jira Bug Summarizer (Bedrock/Claude)

Repository: [YOUR_REPO_NAME]
Required Permissions:
- bedrock:InvokeModel
- bedrock:InvokeModelWithResponseStream (optional)

Requested Role Name: infra-claude-bedrock-ci
Region: us-east-1
Model: us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

EngOps will:
1. Create the IAM role: `arn:aws:iam::ACCOUNT_ID:role/infra-claude-bedrock-ci`
2. Set up OIDC trust relationship for your repository
3. Provide the actual ARN to use in the workflow

---

## Step 2: Update Workflow File

Once you receive the IAM role ARN from EngOps, update `.cloudbees/workflows/jira-bug-report.yaml`:

```yaml
- name: Configure AWS credentials via OIDC
  uses: cloudbees-io/configure-aws-credentials@v1
  with:
    role-to-assume: arn:aws:iam::123456789012:role/infra-claude-bedrock-ci  # Replace with actual ARN
    aws-region: us-east-1
```

---

## Step 3: Configure CloudBees Secrets

Add these secrets to your CloudBees repository:

### Via CloudBees UI:
1. Navigate to your repository settings
2. Go to **Secrets** section
3. Add the following secrets:

| Secret Name | Description | Example Value |
|------------|-------------|---------------|
| `JIRA_EMAIL` | Your Jira email address | `user@cloudbees.com` |
| `JIRA_API_TOKEN` | Jira API token | `ATATT3xFfG...` |

### How to generate Jira API Token:
1. Go to https://id.atlassian.com/manage-profile/security/api-tokens
2. Click **Create API token**
3. Give it a name: "CloudBees CI - Jira Bug Summarizer"
4. Copy the token and save it as `JIRA_API_TOKEN` secret

---

## Step 4: Test the Workflow

### Manual Test:
1. Go to CloudBees CI dashboard
2. Navigate to your workflow: **Jira Bug Aging Report**
3. Click **Run workflow** (workflow_dispatch trigger)
4. Monitor the execution logs

### Expected Output:
```
✅ Successfully authenticated to AWS
✅ Bug report generation completed
📊 Generated Reports:
  - bug_report.json
  - bug_report.txt
```

---

## Step 5: Verify Generated Reports

After successful execution:

1. **Download artifacts:**
   - Go to workflow run details
   - Download `jira-bug-reports-{commit-sha}`

2. **Check the reports:**
   ```bash
   # View JSON report
   cat bug_report.json | jq

   # View text report
   cat bug_report.txt
   ```

---

## Workflow Schedule

The workflow runs automatically:
- **Schedule:** Every Monday at 9:00 AM UTC
- **Manual:** Via workflow_dispatch
- **Auto:** On push to `main` branch (optional - can be removed)

To modify the schedule, edit the `cron` value in the workflow file:
```yaml
on:
  schedule:
    - cron: '0 9 * * 1'  # Minute Hour DayOfMonth Month DayOfWeek
```

**Examples:**
- Daily at 8 AM: `'0 8 * * *'`
- Every weekday at 9 AM: `'0 9 * * 1-5'`
- Twice a day: `'0 9,17 * * *'`

---

## Troubleshooting

### Error: "Unable to locate credentials"
**Solution:** IAM role ARN is incorrect or OIDC trust relationship not configured properly. Contact EngOps.

### Error: "AccessDeniedException: User is not authorized"
**Solution:** IAM role doesn't have `bedrock:InvokeModel` permission. Contact EngOps to add Bedrock permissions.

### Error: "JIRA_EMAIL or JIRA_API_TOKEN not set"
**Solution:** Add secrets to CloudBees repository settings.

### Error: "Model not found"
**Solution:** Verify the model ID is correct and available in `us-east-1` region:
```yaml
BEDROCK_MODEL_ID: us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

---

## Script Configuration

The Python script uses these environment variables (configured in the workflow):

| Variable | Source | Description |
|----------|--------|-------------|
| `JIRA_URL` | Workflow | Jira instance URL |
| `JIRA_EMAIL` | Secret | Jira user email |
| `JIRA_API_TOKEN` | Secret | Jira API token |
| `AWS_REGION` | Workflow | AWS region (us-east-1) |
| `BEDROCK_MODEL_ID` | Workflow | Claude model ID |
| `AWS_PROFILE` | Auto (OIDC) | Handled by OIDC auth |

---

## Customizing the JQL Query

To modify which bugs are fetched, edit `jira_bug_summarizer.py:354`:

```python
# Current query
jql = 'labels = qa_automation AND type = Bug AND status != Done AND status != Rejected'

# Examples:
# All open bugs in a project
jql = 'project = MYPROJECT AND type = Bug AND status in (Open, "In Progress")'

# High priority bugs
jql = 'type = Bug AND priority in (Highest, High) AND status != Done'

# Bugs older than 30 days
jql = 'type = Bug AND created <= -30d AND status != Done'
```

---

## Security Best Practices

1. ✅ **Never commit secrets** to the repository
2. ✅ **Use OIDC** instead of static AWS credentials
3. ✅ **Rotate Jira API tokens** regularly
4. ✅ **Limit IAM role permissions** to only what's needed (bedrock:InvokeModel)
5. ✅ **Review workflow logs** for sensitive data before sharing

---

## Support

- **AWS/OIDC Issues:** Contact EngOps, reference OPS-20629
- **Workflow Issues:** Check CloudBees documentation
- **Script Issues:** Review `jira_bug_summarizer.py` logs

---

## Next Steps

After successful setup:
1. Monitor the first few automated runs
2. Share the reports with your team
3. Consider adding notifications (email, Slack) for report delivery
4. Archive reports for historical analysis
