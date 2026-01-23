# Resources

George uses three main concepts to define and track work. In v1, these map directly to Jira constructs.

| Concept | Jira Implementation |
|---------|---------------------|
| AgentSpec | Engine configuration |
| TaskTemplate | Issue Type + Custom Fields |
| Job | Issue |
| Step | Subtask or Checklist item |

The YAML examples below document the logical structure. In practice, you configure these in Jira's admin UI (issue types, custom fields, screens).

## AgentSpec

Defines what an agent can do - its capabilities, tools, and constraints.

```yaml
apiVersion: george.io/v1
kind: AgentSpec
metadata:
  name: it-provisioning-agent
spec:
  # Container image for this agent type
  image: agents/general-purpose:v1.2

  # LLM configuration
  model: claude-sonnet-4
  maxTokens: 4096

  # Capabilities mounted into the agent
  capabilities:
    - name: slack
      config:
        channels: ["#it-requests", "#it-internal"]
        permissions: [read, write, react]
    - name: jira
      config:
        project: IT
        permissions: [read, create, transition]
    - name: kubectl
      config:
        namespaces: ["employee-*"]
        permissions: [get, list]

  # Resource limits
  maxConcurrentJobs: 5

  # What types of tasks this agent can handle
  taskTypes:
    - it-provisioning
    - access-requests
```

### Fields

| Field | Description |
|-------|-------------|
| `image` | Container image for the agent |
| `model` | LLM model to use |
| `maxTokens` | Max tokens per LLM call |
| `capabilities` | List of capabilities (tools, integrations) |
| `maxConcurrentJobs` | How many jobs this agent type can run simultaneously |
| `taskTypes` | Which task templates this agent can handle |

## TaskTemplate

Defines a type of work that can be performed. Templates are primarily textual - the agent interprets them.

```yaml
apiVersion: george.io/v1
kind: TaskTemplate
metadata:
  name: employee-onboarding
spec:
  description: "Onboard a new employee with all necessary access and equipment"

  # Required parameters when instantiating this template
  parameters:
    - name: employeeName
      type: string
      required: true
    - name: department
      type: string
      required: true
    - name: startDate
      type: date
      required: true
    - name: manager
      type: string
      required: true

  # The goal - agent verifies this at the end
  goal: "Employee has all access and equipment needed to start work on {{ startDate }}"

  # Steps - primarily textual, agent interprets
  steps:
    - name: create-accounts
      description: "Create accounts in all required systems for {{ employeeName }}"
      done_when:
        description: "All accounts created and credentials sent to employee"
        verify:
          - type: jira_ticket_resolved
            project: IT
            label: "accounts-{{ employeeName }}"

    - name: provision-laptop
      description: "Request IT to provision laptop appropriate for {{ department }}"
      done_when:
        description: "IT confirms laptop is ready for pickup"
        verify:
          - type: slack_confirmation
            channel: "#it-requests"
            from_user_in_group: "@it-team"

    - name: verify-access
      description: "Verify {{ employeeName }} can log into all required systems"
      done_when:
        description: "Agent confirms login works for each system"
        verify:
          - type: agent_verification
            prompt: "Verify the employee can access: GitHub, Slack, Email, HR Portal"

    - name: schedule-orientation
      description: "Schedule orientation meeting with {{ manager }}"
      done_when:
        description: "Calendar invite sent and accepted"
        verify:
          - type: calendar_event_exists
            attendees: ["{{ employeeName }}", "{{ manager }}"]

  # Which capabilities are required
  requiredCapabilities:
    - slack
    - jira
    - calendar
```

### Verification Types

| Type | Description |
|------|-------------|
| `jira_ticket_resolved` | Check if a Jira ticket is resolved |
| `slack_confirmation` | Wait for Slack message/reaction |
| `agent_verification` | Agent uses LLM to verify |
| `calendar_event_exists` | Check calendar for event |
| `api_call` | Make API call and check response |
| `cli_exit_code` | Run command and check exit code |

## Job

An instance of a template - actual work to be performed.

```yaml
apiVersion: george.io/v1
kind: Job
metadata:
  name: onboard-alice-2024-01
spec:
  template: employee-onboarding
  parameters:
    employeeName: "Alice Smith"
    department: "Engineering"
    startDate: "2024-02-01"
    manager: "bob@company.com"

  # Optional overrides
  priority: normal
  deadline: "2024-01-31"

  notifications:
    slack:
      channel: "#hr-notifications"
      on: [started, step_completed, blocked, completed, failed]

status:
  state: in_progress
  assignedAgent: it-provisioning-agent-7b4d2
  currentStep: 2
  startedAt: "2024-01-20T10:00:00Z"

  steps:
    - name: create-accounts
      state: completed
      completedAt: "2024-01-20T11:30:00Z"
      evidence:
        - "JIRA IT-1234 resolved"
        - "Credentials email sent"

    - name: provision-laptop
      state: waiting_human
      startedAt: "2024-01-20T11:35:00Z"
      waitingFor:
        type: slack_confirmation
        channel: "#it-requests"
        message: "Waiting for IT team to confirm laptop is ready"

    - name: verify-access
      state: pending

    - name: schedule-orientation
      state: pending
```

## Status States

Steps progress through these states:

```
pending → in_progress → completed
                ↓
            blocked
                ↓
    waiting_human / waiting_agent
                ↓
             failed
```

### State Definitions

| State | Description |
|-------|-------------|
| `pending` | Not yet started |
| `in_progress` | Agent is actively working |
| `completed` | Successfully finished |
| `blocked` | Cannot proceed (capability missing, etc.) |
| `waiting_human` | Waiting for human action |
| `waiting_agent` | Waiting for another agent/sub-job |
| `failed` | Failed after retries exhausted |
| `skipped` | Manually skipped by operator |

### Detailed Status Examples

**Blocked:**
```yaml
status:
  state: blocked
  reason: capability_missing
  detail: "Task requires Confluence API access, but no credentials mounted"
  recoverable: false
```

**Waiting for human:**
```yaml
status:
  state: waiting_human
  reason: external_dependency
  detail: "Waiting for human confirmation in #it-requests"
  waitingFor:
    type: slack_message
    channel: "#it-requests"
    expectedFrom: "@it-team"
  recoverable: true
```

**Failed:**
```yaml
status:
  state: failed
  reason: validation_failed
  detail: "Employee could not log into GitHub after 3 attempts"
  attempts: 3
  lastError: "SSO authentication failed"
  recoverable: maybe  # human should investigate
```

## Notifications

Configurable at template or job level:

```yaml
notifications:
  slack:
    channel: "#hr-notifications"
    events:
      - job_started
      - step_completed
      - job_blocked
      - job_completed
      - job_failed

  webhook:
    url: "https://internal.company.com/george-events"
    events: [all]

  email:
    to: ["hr-team@company.com"]
    events: [job_completed, job_failed]
```

## Context Database

For detailed history too large for Jira comments, George can use a separate context database:

```sql
-- Job execution context
CREATE TABLE job_context (
    id UUID PRIMARY KEY,
    job_name VARCHAR(255) NOT NULL,
    step_name VARCHAR(255),
    timestamp TIMESTAMPTZ NOT NULL,
    event_type VARCHAR(50),  -- 'agent_thought', 'tool_call', 'tool_result', 'human_message'
    content JSONB,
    agent_id VARCHAR(255),
    sequence_number BIGINT
);

-- Evidence and artifacts
CREATE TABLE job_evidence (
    id UUID PRIMARY KEY,
    job_name VARCHAR(255) NOT NULL,
    step_name VARCHAR(255) NOT NULL,
    evidence_type VARCHAR(50),  -- 'screenshot', 'api_response', 'slack_message', 'file'
    content JSONB,
    created_at TIMESTAMPTZ NOT NULL
);
```

**Note:** For v1, Jira comments may be sufficient. The separate DB is for high-volume or detailed tracing needs.
