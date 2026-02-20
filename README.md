# Jira Bug Aging Report

Automatically fetches bugs from Jira, uses AI to summarize comments, calculates bug aging metrics, and generates comprehensive reports.

## Features

- 🔍 Fetches bugs from Jira using JQL queries
- 🤖 AI-powered comment summarization using AWS Bedrock (Claude Sonnet 4.5)
- 📊 Bug aging analysis with color-coded indicators
- 📄 Generates reports in JSON and formatted text
- 🚀 CloudBees CI ready with OIDC authentication
- 📈 Statistical summaries and bug categorization

## Setup

### Local Development Setup

#### 1. Install Dependencies

```bash
cd jira-to-slack
pip install -r requirements.txt
```

#### 2. Configure Credentials

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` with your credentials:

```bash
# Jira Configuration
JIRA_URL=https://cloudbees.atlassian.net
JIRA_EMAIL=your-email@cloudbees.com
JIRA_API_TOKEN=your-jira-api-token

# AWS Bedrock Configuration
AWS_PROFILE=cloudbees-bedrock-claude-infra-bedrock-claude-user
AWS_REGION=us-east-1
BEDROCK_MODEL_ID=us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

#### 3. Get Required Tokens

**Jira API Token:**
1. Go to https://id.atlassian.com/manage-profile/security/api-tokens
2. Click "Create API token"
3. Name it: "Jira Bug Summarizer"
4. Copy the token to `.env` as `JIRA_API_TOKEN`

**AWS Credentials:**
- Ensure you have AWS CLI configured with the CloudBees Bedrock profile
- Your `~/.aws/config` should have the `cloudbees-bedrock-claude-infra-bedrock-claude-user` profile

## Usage

### Local Execution

```bash
python jira_bug_summarizer.py
```

**Generated Files:**
- `bug_report.json` - Structured data with all bug details, comments, and summaries
- `bug_report.txt` - Formatted table report with statistics and aging indicators

### CloudBees CI Integration

The project includes a CloudBees workflow that runs with OIDC authentication (no static credentials needed).

#### Setup CloudBees CI:

1. **Request IAM Role from EngOps:**
   - Reference ticket: [OPS-20629](https://cloudbees.atlassian.net/browse/OPS-20629)
   - Required permission: `bedrock:InvokeModel`
   - Role: `infra-claude-bedrock-ci`

2. **Configure CloudBees Secrets:**
   - `JIRA_EMAIL` - Your Jira email
   - `JIRA_API_TOKEN` - Jira API token

3. **Update Workflow:**
   - Edit `.cloudbees/workflows/jira-bug-report.yaml`
   - Replace `ACCOUNT_ID` with actual AWS account ID (provided by EngOps)

4. **Workflow Triggers:**
   - 📅 **Scheduled:** Every Monday at 9 AM UTC (commented out by default)
   - 🚀 **Manual:** Trigger via CloudBees UI
   - 📌 **Push:** Automatically on push to `main` branch

**For detailed CloudBees setup instructions, see [CLOUDBEES_SETUP.md](CLOUDBEES_SETUP.md)**

## Configuration

### Customize JQL Query

Edit the `run()` method in `jira_bug_summarizer.py` (line 354):

```python
jql = 'labels = qa_automation AND type = Bug AND status != Done AND status != Rejected'
```

**Example JQL queries:**
```python
# All open bugs in a specific project
jql = 'project = MYPROJECT AND type = Bug AND status in (Open, "In Progress")'

# High priority bugs
jql = 'type = Bug AND priority in (Highest, High) AND status != Done'

# Bugs older than 30 days
jql = 'type = Bug AND created <= -30d AND status != Done'
```

### Customize AI Model

Change the model ID in `.env` or workflow:

```bash
BEDROCK_MODEL_ID=us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

Available models:
- `us.anthropic.claude-sonnet-4-5-20250929-v1:0` (default)
- `us.anthropic.claude-opus-4-5-20251101-v1:0`
- `anthropic.claude-3-5-sonnet-20241022-v2:0`

### Adjust Max Results

Edit in `jira_bug_summarizer.py` (line 361):

```python
bugs = self.fetch_bugs(jql, max_results=10)  # Default is 5
```

## Output Examples

### bug_report.txt (Formatted Table)

```
🐛 JIRA BUG SUMMARY REPORT
Report generated on 2026-02-20 12:00 UTC

┌────┬─────────────┬───────────┬─────────────┬──────────┬──────────┬────────────────┬──────────────────────────────────────┐
│ #  │ Ticket ID   │ Days Open │ Last Update │ Status   │ Priority │ Assignee       │ Summary                              │
├────┼─────────────┼───────────┼─────────────┼──────────┼──────────┼────────────────┼──────────────────────────────────────┤
│  1 │ PROJ-123    │ 🟢   5    │ 2026-02-19  │ Open     │ High     │ John Doe       │ Login page crashes on mobile         │
│  2 │ PROJ-124    │ 🟡  15    │ 2026-02-18  │ Progress │ Medium   │ Jane Smith     │ API timeout on large datasets        │
│  3 │ PROJ-125    │ 🟠  45    │ 2026-02-10  │ Open     │ High     │ Unassigned     │ Memory leak in auth module           │
│  4 │ PROJ-126    │ 🔴 120    │ 2025-12-15  │ Blocked  │ Critical │ Bob Wilson     │ Data corruption in migration         │
└────┴─────────────┴───────────┴─────────────┴──────────┴──────────┴────────────────┴──────────────────────────────────────┘

🔗 Links to Tickets:
  1. PROJ-123 - Login page crashes on mobile
  2. PROJ-124 - API timeout on large datasets
  3. PROJ-125 - Memory leak in auth module
  4. PROJ-126 - Data corruption in migration

📊 STATISTICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  • Total Bugs: 4
  • 🔴 Critical Age (90+ days): 1
  • 🟠 Aging (31-90 days): 1
  • 🟡 Active (8-30 days): 1
  • 🟢 Recent (0-7 days): 1
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### bug_report.json (Structured Data)

```json
[
  {
    "bug_key": "PROJ-123",
    "summary": "Login page crashes on mobile",
    "status": "Open",
    "priority": "High",
    "bug_url": "https://cloudbees.atlassian.net/browse/PROJ-123",
    "aging": {
      "created_date": "2026-02-15T10:30:00+00:00",
      "days_open": 5
    },
    "assignee": {
      "name": "John Doe",
      "email": "john.doe@cloudbees.com"
    },
    "comments": {
      "count": 3,
      "summary": "• Root cause: Memory leak in auth module\n• Attempted rollback unsuccessful\n• Waiting for dev environment access",
      "details": [...]
    }
  }
]
```

## Troubleshooting

### "Error: Missing required environment variables"
- **Local:** Ensure `JIRA_EMAIL` and `JIRA_API_TOKEN` are set in `.env`
- **CloudBees CI:** Verify secrets are configured in repository settings

### "Error fetching bugs"
- Check Jira credentials and URL
- Verify JQL query syntax at https://cloudbees.atlassian.net/issues
- Ensure user has permission to view bugs
- Check if Jira API token is valid

### "Unable to locate credentials" (AWS)
- **Local:** Verify AWS CLI profile exists: `aws configure list --profile cloudbees-bedrock-claude-infra-bedrock-claude-user`
- **CloudBees CI:** Ensure IAM role ARN is correct and OIDC trust is configured

### "AccessDeniedException: User is not authorized" (Bedrock)
- IAM role missing `bedrock:InvokeModel` permission
- Contact EngOps to add Bedrock permissions to the role
- Verify model ID is correct and available in `us-east-1`

### "Model not found"
- Check model availability in AWS Bedrock console
- Verify model ID format: `us.anthropic.claude-sonnet-4-5-20250929-v1:0`
- Ensure Bedrock is enabled in `us-east-1` region

## Project Structure

```
jira-to-slack/
├── .cloudbees/
│   └── workflows/
│       └── jira-bug-report.yaml      # CloudBees CI workflow
├── jira_bug_summarizer.py            # Main Python script
├── requirements.txt                  # Python dependencies
├── CLOUDBEES_SETUP.md                # Detailed CloudBees setup guide
├── README.md                         # This file
├── .env.example                      # Environment template (local dev)
├── .env.cloudbees                    # Environment reference (CI)
├── .env                              # Your local credentials (gitignored)
├── .gitignore                        # Git ignore rules
└── bug_report.* (generated)          # Output files (JSON + TXT)
```

## Key Technologies

- **Jira REST API** - Bug data fetching
- **AWS Bedrock** - Claude AI for comment summarization
- **CloudBees CI** - Automation platform with OIDC authentication
- **Python 3.13** - Core runtime

## Related Documentation

- [CLOUDBEES_SETUP.md](CLOUDBEES_SETUP.md) - Complete CloudBees CI setup guide
- [OPS-20629](https://cloudbees.atlassian.net/browse/OPS-20629) - IAM role setup ticket

## Notes

- **Slack Integration:** Currently commented out in the code. Can be re-enabled if needed.
- **Authentication:** Uses OIDC in CloudBees CI (no static credentials), AWS profiles for local dev.
- **Scheduled Runs:** Schedule is commented out by default in workflow. Uncomment to enable weekly reports.
