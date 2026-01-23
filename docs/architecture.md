# Architecture

Technical design for George - the GitOps-driven LLM agent reconciliation engine.

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         Git Repository                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ AgentSpecs  │  │  Templates  │  │   Jobs (instances)  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────┬───────────────────────────────┘
                              │ sync
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      George Engine                           │
│  ┌─────────────────┐  ┌─────────────┐  ┌─────────────────┐  │
│  │  Reconciliation │  │   Scheduler │  │    Dashboard    │  │
│  │      Loop       │  │             │  │       API       │  │
│  └────────┬────────┘  └──────┬──────┘  └─────────────────┘  │
│           │                  │                               │
│           ▼                  ▼                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              State Store (CRDs or DB)                   ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────┬───────────────────────────────┘
                              │ spawn/manage
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Agent Pods                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Agent 1   │  │   Agent 2   │  │   Agent N   │          │
│  │ (job: xyz)  │  │ (job: abc)  │  │ (job: ...)  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

## Resource Types

### AgentSpec

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

### TaskTemplate

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
        # Optional: structured verification
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

  # Which agent types can run this
  requiredCapabilities:
    - slack
    - jira
    - calendar
```

### Job

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

  # Optional: override or add constraints
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

### Detailed Status Fields

```yaml
status:
  state: blocked
  reason: capability_missing
  detail: "Task requires Confluence API access, but no credentials mounted"
  recoverable: false

# vs

status:
  state: waiting_human
  reason: external_dependency
  detail: "Waiting for human confirmation in #it-requests"
  waitingFor:
    type: slack_message
    channel: "#it-requests"
    expectedFrom: "@it-team"
  recoverable: true  # will resume when signal received

# vs

status:
  state: failed
  reason: validation_failed
  detail: "Employee could not log into GitHub after 3 attempts"
  attempts: 3
  lastError: "SSO authentication failed"
  recoverable: maybe  # human should investigate
```

## Agent Lifecycle

### Spawn

1. Engine sees Job needs processing
2. Engine selects suitable AgentSpec based on `requiredCapabilities`
3. Engine spawns agent pod with:
   - Job reference
   - Capability credentials mounted
   - Context DB connection

### Execute

1. Agent reads Job spec from CRD
2. Agent reads conversation history from context DB
3. Agent determines current state: "where am I in this job?"
4. Agent decides next action based on LLM interpretation
5. Agent executes (idempotently)
6. Agent updates Job status CRD
7. Agent logs to context DB
8. Repeat until done or blocked

### Terminate

- **Completed**: Agent marks Job as completed, pod terminates
- **Blocked**: Agent marks Job as blocked, pod may terminate (engine will retry later)
- **Crash**: Engine detects pod died, spawns new agent, which reconstructs state

## Context Database Schema

Stores detailed conversation history and evidence (too large for CRDs).

```sql
-- Job execution context
CREATE TABLE job_context (
    id UUID PRIMARY KEY,
    job_name VARCHAR(255) NOT NULL,
    step_name VARCHAR(255),
    timestamp TIMESTAMPTZ NOT NULL,

    -- What happened
    event_type VARCHAR(50),  -- 'agent_thought', 'tool_call', 'tool_result', 'human_message', 'state_change'
    content JSONB,

    -- For reconstruction
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

## Dashboard View

The engine provides a dashboard showing:

```
┌─────────────────────────────────────────────────────────────┐
│ Job: onboard-alice-2024-01                                  │
│ Template: employee-onboarding                               │
│ Status: in_progress (step 2/4)                              │
├─────────────────────────────────────────────────────────────┤
│ ✓ Step 1: Create accounts          [completed]              │
│   └─ Evidence: JIRA IT-1234, credentials sent               │
│   └─ Duration: 1h 30m                                       │
│                                                             │
│ ◐ Step 2: Provision laptop         [waiting_human]          │
│   └─ Waiting: IT confirmation in #it-requests               │
│   └─ Waiting since: 2h 15m                                  │
│                                                             │
│ ○ Step 3: Verify access            [pending]                │
│                                                             │
│ ○ Step 4: Schedule orientation     [pending]                │
├─────────────────────────────────────────────────────────────┤
│ Agent: it-provisioning-agent-7b4d2                          │
│ Started: 2024-01-20 10:00 UTC                               │
│ Deadline: 2024-01-31                                        │
└─────────────────────────────────────────────────────────────┘
```

## Notifications / Hooks

Configurable at engine level, job level, or task template level:

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

## Security Model

### Capability Scoping

Agents only get credentials for capabilities they need:

```yaml
# AgentSpec defines what it CAN have
capabilities:
  - slack: {channels: [...], permissions: [...]}
  - jira: {project: IT}

# Job's template defines what it NEEDS
requiredCapabilities:
  - slack
  - jira

# Engine mounts intersection at runtime
```

### Principle of Least Privilege

- Agents can't access capabilities not in their AgentSpec
- Agents can't access capabilities not required by the task template
- Credentials are mounted read-only and scoped

### Audit Trail

Everything is logged:
- Git commits for all definition changes
- CRD status changes with timestamps
- Context DB for detailed execution history
- Agent pod logs for debugging
