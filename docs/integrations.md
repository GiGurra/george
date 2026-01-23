# Integrations

George integrates with external systems for storage, communication, and interaction.

## Jira (Primary Storage + Visualization)

Jira serves as the primary storage and visualization layer for George. This gives us:

- **Issue tracking** - Jobs as issues, steps as subtasks or checklists
- **Workflow states** - Built-in status transitions
- **Comments** - Agent activity log, human interactions
- **Attachments** - Evidence and artifacts
- **Dashboards** - Built-in visualization, boards, timelines
- **Mobile app** - Visibility on the go
- **Audit trail** - Full history out of the box

### Jira Service Accounts

Each agent template gets its own Jira service account:

| Agent Template | Jira Service Account |
|----------------|---------------------|
| it-provisioning-agent | `george-it-provisioning` |
| onboarding-agent | `george-onboarding` |
| security-review-agent | `george-security-review` |

**Why one per template?**

- Templates define permissions/capabilities - matches service account scoping
- Free tier gives 5 service accounts (enough to prototype)
- Comments show which template performed the action
- Agent instances sign their ID in comments for traceability

**Service account limits:**

- Free: 5 service accounts per organization
- Guard Standard: up to 250
- Guard Enterprise: up to 1,000

### Agent Identity in Comments

Agents sign their instance ID in comments:

```
[agent: it-provisioning-agent-7b4d2]

Checked Slack channel #it-requests for laptop confirmation.
Found message from @sarah.it at 14:32: "Laptop ready for Alice, pickup at reception"

Marking step as complete.
```

This provides:
- **Template identity** (who posted) - from Jira service account
- **Instance identity** (which run) - from signed comment

### Jira Mapping

| George Concept | Jira Concept |
|----------------|--------------|
| Job | Issue (Epic or Story) |
| Job parameters | Custom fields or description |
| Step | Subtask or checklist item |
| Step status | Subtask status / checkbox |
| Evidence | Comments + attachments |
| Agent activity | Comments (signed by agent) |
| Notifications | Jira automation rules |

### Example Jira Structure

```
GEORGE-123: Onboard Alice Smith (Epic)
├── Status: In Progress
├── Description: Employee onboarding for Engineering dept
├── Custom Fields:
│   ├── Employee: Alice Smith
│   ├── Department: Engineering
│   ├── Start Date: 2024-02-01
│   └── Manager: bob@company.com
├── Subtasks:
│   ├── GEORGE-124: Create accounts [Done]
│   ├── GEORGE-125: Provision laptop [In Progress]
│   ├── GEORGE-126: Verify access [To Do]
│   └── GEORGE-127: Schedule orientation [To Do]
└── Comments:
    ├── [george-onboarding] Job started by engine
    ├── [agent: onboarding-agent-7b4d2] Created GitHub account...
    ├── [agent: onboarding-agent-7b4d2] Created Slack account...
    └── [agent: onboarding-agent-7b4d2] Waiting for IT confirmation...
```

### Jira API Notes

Service accounts use scoped API tokens with a different URL format:

```bash
# Normal user API
curl https://company.atlassian.net/rest/api/3/issue/GEORGE-123

# Service account API (scoped tokens)
curl https://api.atlassian.com/ex/jira/{cloudId}/rest/api/3/issue/GEORGE-123
```

## Slack (Human-in-the-Loop)

Slack is used for human communication and confirmations.

### Use Cases

- **Notifications** - Job started, completed, failed, blocked
- **Confirmations** - Wait for human to confirm work is done
- **Approvals** - Request approval before sensitive steps
- **Questions** - Agent asks clarifying questions
- **Delegation** - Agent delegates tasks to humans

### Agent Communication

```
#it-requests channel:

[george-onboarding] 🔔 New request
  Job: GEORGE-123 - Onboard Alice Smith
  Step: Provision laptop

  Please provision a laptop for Alice Smith (Engineering).
  React with ✅ when ready for pickup.

  View in Jira: https://company.atlassian.net/browse/GEORGE-123

---

@sarah.it: Laptop ready, serial #ABC123, at reception desk ✅

---

[george-onboarding] ✓ Confirmed by @sarah.it
  Proceeding to next step.
```

### Waiting for Signals

Agents can wait for specific Slack signals:

```yaml
done_when:
  verify:
    - type: slack_confirmation
      channel: "#it-requests"
      from_user_in_group: "@it-team"
      signal: reaction  # or "message", "thread_reply"
```

## CLI (`george`)

Command-line interface for interacting with the George engine.

### Installation

```bash
# Via Go
go install github.com/gigurra/george/cmd/george@latest

# Or download binary
curl -L https://github.com/gigurra/george/releases/latest/download/george-$(uname -s)-$(uname -m) -o george
chmod +x george
```

### Commands

```bash
# Job management
george jobs                          # List all jobs
george jobs --status in_progress     # Filter by status
george get job/GEORGE-123            # Get job details
george logs job/GEORGE-123           # View job activity log

# Job control
george run template/onboarding \
  --param employeeName="Alice Smith" \
  --param department="Engineering"   # Create and start job

george pause job/GEORGE-123          # Pause job
george resume job/GEORGE-123         # Resume job
george abort job/GEORGE-123          # Abort job

# Interactive mode
george attach job/GEORGE-123         # Attach to running job
george run job/GEORGE-123 --interactive  # Start in interactive mode

# Debugging
george dry-run job/GEORGE-123        # Preview next step
george breakpoint job/GEORGE-123 --before verify-access

# Overrides
george override job/GEORGE-123 step/provision-laptop \
  --action complete \
  --reason "Confirmed via phone"

# Agent management
george agents                        # List running agents
george logs agent/onboarding-agent-7b4d2  # Agent logs

# Templates
george templates                     # List templates
george get template/onboarding       # View template details
```

### Configuration

```yaml
# ~/.george/config.yaml
engine:
  url: http://localhost:8080
  # or
  url: https://george.internal.company.com

jira:
  url: https://company.atlassian.net
  project: GEORGE

defaults:
  output: table  # or "json", "yaml"
```

## MCP / Claude Code Skills

Interact with George directly from Claude Code or other MCP-compatible tools.

### MCP Server

George can expose an MCP server for AI assistants:

```yaml
# george-config.yaml
mcp:
  enabled: true
  port: 8081
```

### Available Tools

When connected, Claude Code gets these tools:

```
george_list_jobs      - List all jobs with optional filters
george_get_job        - Get detailed job information
george_create_job     - Create a new job from template
george_pause_job      - Pause a running job
george_resume_job     - Resume a paused job
george_override_step  - Manually override a step status
george_inject_message - Send a message to a running agent
george_dry_run        - Preview what agent would do next
```

### Example Usage in Claude Code

```
User: What George jobs are currently running?

Claude: [calls george_list_jobs with status="in_progress"]

There are 3 jobs currently running:
- GEORGE-123: Onboard Alice Smith (step 2/4)
- GEORGE-124: Security review for PR #456 (step 1/3)
- GEORGE-125: Provision dev environment (step 3/5)

User: Pause the onboarding job, I need to check something

Claude: [calls george_pause_job with job_id="GEORGE-123"]

Done. Job GEORGE-123 is now paused. The agent will stop after
completing its current action. Use "resume" when ready to continue.
```

### Claude Code Skill

A dedicated skill for George interaction:

```yaml
# george.skill.yaml
name: george
description: Interact with George agent orchestration engine
triggers:
  - "george"
  - "list jobs"
  - "check agents"

tools:
  - george_list_jobs
  - george_get_job
  - george_create_job
  # ... etc
```

## Webhooks (Outbound)

George can send webhooks for external integrations:

```yaml
notifications:
  webhook:
    url: "https://internal.company.com/george-events"
    events: [job_started, job_completed, job_failed, step_blocked]
    headers:
      Authorization: "Bearer ${WEBHOOK_TOKEN}"
```

### Webhook Payload

```json
{
  "event": "step_completed",
  "timestamp": "2024-01-20T14:30:00Z",
  "job": {
    "id": "GEORGE-123",
    "template": "employee-onboarding",
    "status": "in_progress"
  },
  "step": {
    "name": "create-accounts",
    "status": "completed",
    "evidence": ["JIRA IT-1234 resolved", "Credentials sent"]
  },
  "agent": {
    "id": "onboarding-agent-7b4d2",
    "template": "onboarding-agent"
  }
}
```
