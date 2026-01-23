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

## Human Intervention

Humans remain in control. The engine supports several intervention mechanisms.

### Manual Override

Any step can be manually marked as completed, skipped, or unblocked - useful when:

- A verification check has a bug
- External conditions changed
- The agent is stuck but the work was actually done

```yaml
# Via CLI
george override job/onboard-alice-2024-01 step/provision-laptop \
  --action complete \
  --reason "Laptop confirmed via phone call, Slack was down" \
  --by "johan@company.com"

# Results in status update
status:
  steps:
    - name: provision-laptop
      state: completed
      completedAt: "2024-01-20T14:00:00Z"
      override:
        action: manual_complete
        reason: "Laptop confirmed via phone call, Slack was down"
        by: "johan@company.com"
        at: "2024-01-20T14:00:00Z"
```

### Manual Unblock

When a step is blocked (e.g., capability missing), humans can:

1. **Fix and retry** - Add the missing capability, then resume
2. **Skip** - Mark step as skipped, continue to next
3. **Override complete** - Mark as done despite the block
4. **Abort** - Cancel the entire job

```yaml
# Unblock by skipping
george unblock job/onboard-alice-2024-01 step/verify-access \
  --action skip \
  --reason "Verification will be done manually post-onboarding"
```

### Approval Gates

Steps can require explicit human approval before proceeding:

```yaml
steps:
  - name: delete-old-accounts
    description: "Delete accounts from previous system"
    requires_approval:
      from: ["security-team@company.com", "manager"]
      message: "Please confirm old accounts should be deleted"
    done_when:
      description: "Old accounts deleted"
```

## Interactive Mode

For debugging, demos, or high-stakes jobs, run in interactive mode.

### Capabilities

- **Pause/Resume** - Stop agent at any point, resume later
- **Step-through** - Execute one step at a time, wait for human approval
- **Watch** - Real-time streaming of agent thoughts and actions
- **Inject** - Send messages/hints to the agent mid-execution
- **Rollback** - Undo last step (if step supports compensation)

### CLI Usage

```bash
# Start job in interactive mode
george run job/onboard-alice-2024-01 --interactive

# Attach to running job
george attach job/onboard-alice-2024-01

# Interactive commands
> pause                    # Pause agent
> resume                   # Resume agent
> step                     # Execute one step, then pause
> inject "Check the #it-backup channel instead"
> status                   # Show current state
> history                  # Show recent agent actions
> abort                    # Cancel job
```

### Dashboard Integration

Interactive mode also available in dashboard:

```
┌─────────────────────────────────────────────────────────────┐
│ Job: onboard-alice-2024-01                    [INTERACTIVE] │
├─────────────────────────────────────────────────────────────┤
│ Agent thinking...                                           │
│ > "I need to check if the laptop request was acknowledged   │
│ >  in #it-requests. Let me search for recent messages..."   │
│                                                             │
│ [Pause] [Step] [Inject Message] [Abort]                     │
├─────────────────────────────────────────────────────────────┤
│ ✓ Step 1: Create accounts          [completed]              │
│ ▶ Step 2: Provision laptop         [in_progress]            │
│   └─ Agent searching Slack...                               │
└─────────────────────────────────────────────────────────────┘
```

### Breakpoints

Set breakpoints to pause execution at specific points:

```yaml
# In job spec
spec:
  breakpoints:
    - before: verify-access      # Pause before this step starts
    - after: create-accounts     # Pause after this step completes
    - on_subtask: true           # Pause when agent creates sub-tasks
    - on_error: true             # Pause on any error (instead of auto-retry)
```

```bash
# Via CLI
george breakpoint job/onboard-alice --before verify-access
george breakpoint job/onboard-alice --clear  # Remove all breakpoints
```

When a breakpoint hits, the job pauses and notifies the operator:

```
┌─────────────────────────────────────────────────────────────┐
│ Job: onboard-alice-2024-01               [PAUSED:BREAKPOINT]│
├─────────────────────────────────────────────────────────────┤
│ ⏸ Breakpoint hit: before step "verify-access"              │
│                                                             │
│ [Resume] [Step Once] [Dry-Run Next] [Inspect] [Abort]       │
└─────────────────────────────────────────────────────────────┘
```

### Dry-Run Next Step

Before executing, ask the agent to explain what it *would* do:

```bash
george dry-run job/onboard-alice-2024-01

# Output:
Step: verify-access
Agent analysis:
  "I would verify access for Alice Smith to the following systems:
   - GitHub: Check if alice.smith can access github.com/company
   - Slack: Verify membership in required channels
   - Email: Confirm O365 login works
   - HR Portal: Test SSO authentication

   I would do this by:
   1. Using the GitHub API to check org membership
   2. Using Slack API to verify channel membership
   3. For Email and HR Portal, I would need to ask Alice to confirm
      or use a test credential if available

   Potential issues:
   - I don't have Alice's credentials to test login directly
   - May need to ask Alice to verify herself via Slack DM"

Estimated actions: 4 API calls, 1 Slack DM
Estimated tokens: ~2000

[Execute] [Skip] [Modify Step] [Abort]
```

This helps operators:
- Understand agent reasoning before committing
- Catch misunderstandings early
- Estimate costs before execution

### Step Preview in Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│ Next Step Preview (dry-run)                                 │
├─────────────────────────────────────────────────────────────┤
│ Step: verify-access                                         │
│                                                             │
│ Agent would:                                                │
│  • Call GitHub API: GET /orgs/company/members/alice.smith   │
│  • Call Slack API: Check #engineering membership            │
│  • Send DM to Alice: "Please confirm you can log into..."   │
│                                                             │
│ Confidence: High                                            │
│ Estimated tokens: ~2000                                     │
│ Potential issues: None detected                             │
├─────────────────────────────────────────────────────────────┤
│ [Execute] [Execute with Modifications] [Skip] [Edit Step]   │
└─────────────────────────────────────────────────────────────┘
```

## Sub-Tasks

Steps can spawn sub-tasks for complex work. Sub-tasks are tracked as nested structures within the job.

### Modeling

```yaml
status:
  steps:
    - name: create-accounts
      state: in_progress
      subtasks:
        - name: create-github-account
          state: completed
          evidence: "Invite sent to alice@company.com"
        - name: create-slack-account
          state: completed
          evidence: "User alice.smith created"
        - name: create-jira-account
          state: in_progress
        - name: create-email-account
          state: pending
```

### Agent-Created Sub-Tasks

Agents can dynamically create sub-tasks as they discover work:

```yaml
# Agent discovers additional systems needed for Engineering dept
subtasks:
  - name: create-aws-account
    state: pending
    created_by: agent
    reason: "Engineering department requires AWS access"
```

### Parallel Sub-Tasks

Sub-tasks can run in parallel when independent:

```yaml
steps:
  - name: create-accounts
    parallel_subtasks: true  # Agent can work on multiple simultaneously
```

## Retry Policies

Configurable at template or job level.

```yaml
spec:
  retry:
    # Step-level retries
    maxAttempts: 3
    backoff:
      initial: 30s
      multiplier: 2
      max: 10m

    # What to retry on
    retryOn:
      - transient_error
      - timeout
      - agent_crash

    # What NOT to retry on
    noRetryOn:
      - validation_failed
      - capability_missing
      - manual_abort
```

### Retry Status

```yaml
status:
  steps:
    - name: verify-access
      state: failed
      attempts:
        - attempt: 1
          at: "2024-01-20T12:00:00Z"
          error: "GitHub SSO timeout"
        - attempt: 2
          at: "2024-01-20T12:01:00Z"
          error: "GitHub SSO timeout"
        - attempt: 3
          at: "2024-01-20T12:03:00Z"
          error: "GitHub SSO timeout"
      finalError: "Max attempts exceeded"
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

## Live Job Modifications

Jobs can be modified while running. The engine reconciles the new desired state with current progress.

**Note:** This is an advanced feature - may not be in v1.

### Use Cases

- **Add steps** - Discovered additional work needed mid-job
- **Remove steps** - Step no longer relevant due to external changes
- **Modify parameters** - Correct a typo or update requirements
- **Insert steps** - Add a step between existing steps
- **Change step order** - Reorder remaining pending steps

### Constraints

- **Completed steps** - Cannot be undone (but can add compensation steps)
- **In-progress steps** - Must complete or be cancelled before modification takes effect
- **Dependencies** - Engine validates new graph doesn't break dependencies

### Example: Adding a Step Mid-Job

```bash
# Original job has 4 steps, currently on step 2
george modify job/onboard-alice-2024-01 \
  --add-step-after provision-laptop \
  --step-name "security-training" \
  --step-description "Ensure Alice completes security training" \
  --done-when "Training completion certificate received"

# Job now has 5 steps:
# ✓ Step 1: create-accounts       [completed]
# ◐ Step 2: provision-laptop      [in_progress]
# ○ Step 3: security-training     [pending] ← NEW
# ○ Step 4: verify-access         [pending]
# ○ Step 5: schedule-orientation  [pending]
```

### Modification Status

```yaml
status:
  modifications:
    - at: "2024-01-20T15:00:00Z"
      by: "johan@company.com"
      type: add_step
      detail: "Added security-training after provision-laptop"
      reason: "Compliance requirement discovered"
```

### Via Git (GitOps Style)

Modifications can also come through Git:

1. Edit the Job YAML in Git
2. Commit and push
3. Engine detects change
4. Engine validates modification is safe
5. Engine applies modification to running job

```yaml
# The job spec can include modification policy
spec:
  allowLiveModifications:
    addSteps: true
    removeSteps: pending_only  # Can only remove pending steps
    reorderSteps: true
    modifyParameters: false    # Lock parameters after start
```

### Validation

Engine validates before applying:

```
Modification rejected:
  Cannot remove step "create-accounts" - already completed.
  Cannot reorder step "verify-access" before "provision-laptop" -
    verify-access depends on provision-laptop output.
```

## Future Considerations

### Template Versioning

What happens when a template changes while jobs are running?

Options:
1. **Pin on start** - Job uses template version from when it started
2. **Live update** - Job picks up template changes (risky)
3. **Manual migration** - Operator decides per-job

Likely: Pin on start, with option to manually migrate.

### Rollback / Compensation

If step 3 fails, should we undo steps 1-2?

```yaml
steps:
  - name: create-accounts
    compensate:
      description: "Delete created accounts"
      # Agent interprets this if rollback triggered
```

This is complex - not all actions are reversible. Probably a v2 feature.

### Concurrency Control

What if two jobs need the same resource (same Slack channel, same human)?

```yaml
spec:
  locks:
    - resource: "slack:#it-requests"
      scope: exclusive  # Only one job can use at a time
```

### Cost Tracking

Track LLM token usage per job for enterprise billing:

```yaml
status:
  costs:
    totalTokens: 45000
    estimatedCost: "$0.45"
    byStep:
      create-accounts: 12000
      provision-laptop: 8000
```

### Dry-Run Mode

See what an agent *would* do without executing:

```bash
george run job/onboard-alice --dry-run
```

Agent narrates its plan without taking actions.

### Agent-to-Agent Communication

Jobs spawning sub-jobs handled by different agents:

```yaml
steps:
  - name: request-security-review
    delegate_to:
      template: security-review
      parameters:
        target: "{{ employeeName }}"
      wait: true  # Block until sub-job completes
```

### Recursive Reconciliation

Agents that spawn other tracked agents. The engine would manage:
- Parent-child job relationships
- Cascading cancellation
- Aggregated status views
