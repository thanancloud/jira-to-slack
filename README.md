# Jira Bug Aging Report

Automatically fetches bugs from Jira, uses AI to summarize comments, calculates bug aging metrics, and generates comprehensive reports.

## Features

- 🔍 Fetches bugs from Jira using customizable JQL queries (via environment variable)
- 🤖 AI-powered comment summarization using AWS Bedrock (Claude Sonnet 4.5)
- 📊 Bug aging analysis with color-coded indicators
- 🎯 Priority-based sorting (Highest to Lowest)
- 👥 Atlassian Team field extraction (displays team names like "CBP Ninja", "CBP Core UI")
- 💬 Separate comment summary table with AI-generated summaries
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
JIRA_JQL="labels = qa_automation AND type = Bug AND status != Done AND status != Rejected"

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

Set the `JIRA_JQL` environment variable in `.env`:

```bash
JIRA_JQL="labels = qa_automation AND type = Bug AND status != Done AND status != Rejected"
```

**Example JQL queries:**
```bash
# All open bugs in a specific project
JIRA_JQL="project = MYPROJECT AND type = Bug AND status in (Open, 'In Progress')"

# High priority bugs
JIRA_JQL="type = Bug AND priority in (Highest, High) AND status != Done"

# Bugs older than 30 days
JIRA_JQL="type = Bug AND created <= -30d AND status != Done"
```

If `JIRA_JQL` is not set, it defaults to:
```bash
"labels = qa_automation AND type = Bug AND status != Done AND status != Rejected"
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

Edit the `fetch_bugs` method signature in `jira_bug_summarizer.py`:

```python
def fetch_bugs(self, jql: str, max_results: int = 20) -> List:
    # Change default from 20 to your desired number
```

Or when calling:
```python
bugs = self.fetch_bugs(self.jql_query, max_results=50)
```

### Team Field Configuration

The script automatically detects and uses the **Atlassian Team field** (`customfield_12000`) from your Jira instance. This field:

- **Schema Type:** `team`
- **Custom Type:** `com.atlassian.jira.plugin.system.customfieldtypes:atlassian-team`
- **Returns:** Team names like "CBP Ninja", "CBP Core UI", "CBP CD-RO Team"

**How it works:**
1. On startup, the script searches all Jira custom fields for the Atlassian Team field
2. Automatically includes this field when fetching issues
3. Extracts the team name from the `PropertyHolder` object
4. Falls back to components/labels if no team is assigned

**If your Jira instance uses a different team field structure:**
- The script will print: `"Atlassian Team field not found, will use components/labels"`
- Modify `get_team_field_id()` method to match your team field schema

## Output Examples

### bug_report.txt (Formatted Table)

```
🐛 JIRA BUG SUMMARY REPORT
Report generated on 2026-02-24 10:55 UTC

┌────┬─────────────┬───────────┬─────────────┬──────────┬──────────┬────────────────────┬────────────────┐
│ #  │ Ticket ID   │ Days Open │ Last Update │ Status   │ Priority │ Teams              │ Assignee       │
├────┼─────────────┼───────────┼─────────────┼──────────┼──────────┼────────────────────┼────────────────┤
│  1 │ CBP-33505   │ 🟢   4     │ 2026-02-23 │ To Do    │ Medium   │ CBP Core UI        │ Unassigned     │
│  2 │ CBP-33465   │ 🟢   4     │ 2026-02-23 │ To Do    │ Medium   │ CBP CD-RO Team     │ Alexey Ivanov  │
│  3 │ CBP-33206   │ 🟡  11     │ 2026-02-12 │ To Do    │ Medium   │ CBP CD-RO Team     │ Unassigned     │
│  4 │ CBP-33198   │ 🟡  11     │ 2026-02-23 │ To Do    │ Medium   │ CBP Ninja          │ Unassigned     │
│  5 │ CBP-31527   │ 🟡  26     │ 2026-01-28 │ To Do    │ Medium   │ CBP Ninja          │ Unassigned     │
└────┴─────────────┴───────────┴─────────────┴──────────┴──────────┴────────────────────┴────────────────┘

💬 COMMENT SUMMARIES
┌────┬─────────────┬────────────────────────────────────────────────────────────────────────────────────────┐
│ #  │ Ticket ID   │ AI-Generated Comment Summary                                                           │
├────┼─────────────┼────────────────────────────────────────────────────────────────────────────────────────┤
│  1 │ CBP-33505   │ ## Summary for CBP-33505                                                               │
│    │             │                                                                                        │
│    │             │ - **Root Cause**: Organization creation working but button lacks feedback during       │
│    │             │   processing, allowing multiple submissions                                            │
│    │             │ - **Solution Recommended**: Disable button after click + loading spinner               │
│    │             │ - **Current Status**: Converted from bug to task, UX pattern approved                  │
├────┼─────────────┼────────────────────────────────────────────────────────────────────────────────────────┤
│  2 │ CBP-33465   │ ## Summary for CBP-33465                                                               │
│    │             │                                                                                        │
│    │             │ - **Root Cause**: New runs not appearing in Application Release view                   │
│    │             │ - **Current Status**: Issue confirmed on Pre-prod, affects multiple views              │
└────┴─────────────┴────────────────────────────────────────────────────────────────────────────────────────┘

🔗 Links to Tickets:
  1. CBP-33505 - Create Org UX Experience
  2. CBP-33465 - QA: Application release run details failing to load
  3. CBP-33206 - [QA] In release drawer shows that version doesn't exist
  4. CBP-33198 - Mismatch between workflow run duration and DORA metrics
  5. CBP-31527 - AWS CodeDeploy > The workflow execution was unsuccessful

📊 STATISTICS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  • Total Bugs: 5
  • 🔴 Critical Age (90+ days): 0
  • 🟠 Aging (31-90 days): 0
  • 🟡 Active (8-30 days): 3
  • 🟢 Recent (0-7 days): 2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Note: Bugs are sorted by priority (Highest to Lowest)
      Teams displayed from Atlassian Team field (e.g., "CBP Ninja", "CBP Core UI")
```

### bug_report.json (Structured Data)

```json
[
  {
    "bug_key": "CBP-33505",
    "summary": "Create Org UX Experience",
    "status": "To Do",
    "priority": "Medium",
    "bug_url": "https://cloudbees.atlassian.net/browse/CBP-33505",
    "aging": {
      "created_date": "2026-02-19T22:29:42.367-0800",
      "days_open": 4
    },
    "last_updated": "2026-02-23T01:13:48.701-0800",
    "team": {
      "team_name": "CBP Core UI",
      "components": [],
      "labels": ["qa_automation"]
    },
    "reporter": {
      "name": "Hari Prasath",
      "email": "hkrishnamurthy@cloudbees.com"
    },
    "assignee": {
      "name": "Unassigned",
      "email": null
    },
    "comments": {
      "count": 4,
      "summary": "## Summary for CBP-33505\n\n- **Root Cause**: Organization creation working but button lacks feedback...\n- **Solution Recommended**: Disable button after click + loading spinner\n- **Current Status**: Converted from bug to task, UX pattern approved",
      "details": [
        {
          "author": "Patti McLetchie",
          "created": "2026-02-20T05:56:40.598-0800",
          "body": "can you please include the actual result and expected result..."
        }
      ]
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

### Team field showing "N/A" or labels instead of team names
- Check console output: Look for `"Found Atlassian Team field: Team (ID: customfield_12000)"`
- If you see `"Atlassian Team field not found"`, your Jira may use a different team field
- Verify issues have teams assigned in Jira (check the "Team" field in the issue details)
- If using a custom team field, modify the `get_team_field_id()` method to match your schema

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
- **Team Field:** Automatically extracts team names from Atlassian Team field (`customfield_12000`). Falls back to components/labels if team field is not found or not populated.
- **Sorting:** Bugs are automatically sorted by priority (Highest → High → Medium → Low → Lowest) in all reports.
- **Comment Summaries:** AI-generated summaries appear in a separate table below the main bug details table.
