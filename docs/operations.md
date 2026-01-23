# Operations

Human intervention, debugging, and operational controls for George jobs.

## Human Intervention

Humans remain in control. The engine supports several intervention mechanisms.

### Manual Override

Any step can be manually marked as completed, skipped, or unblocked - useful when:

- A verification check has a bug
- External conditions changed
- The agent is stuck but the work was actually done

```bash
george override job/GEORGE-123 step/provision-laptop \
  --action complete \
  --reason "Laptop confirmed via phone call, Slack was down" \
  --by "johan@company.com"
```

Results in status update:

```yaml
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

```bash
george unblock job/GEORGE-123 step/verify-access \
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
george run job/GEORGE-123 --interactive

# Attach to running job
george attach job/GEORGE-123

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

Interactive mode also available in dashboard (Jira or future custom UI):

```
┌─────────────────────────────────────────────────────────────┐
│ Job: GEORGE-123                               [INTERACTIVE] │
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

## Debugging

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
george breakpoint job/GEORGE-123 --before verify-access
george breakpoint job/GEORGE-123 --clear  # Remove all breakpoints
```

When a breakpoint hits, the job pauses and notifies the operator:

```
┌─────────────────────────────────────────────────────────────┐
│ Job: GEORGE-123                          [PAUSED:BREAKPOINT]│
├─────────────────────────────────────────────────────────────┤
│ ⏸ Breakpoint hit: before step "verify-access"              │
│                                                             │
│ [Resume] [Step Once] [Dry-Run Next] [Inspect] [Abort]       │
└─────────────────────────────────────────────────────────────┘
```

### Dry-Run Next Step

Before executing, ask the agent to explain what it *would* do:

```bash
george dry-run job/GEORGE-123

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

### Step Preview

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
    maxAttempts: 3
    backoff:
      initial: 30s
      multiplier: 2
      max: 10m

    retryOn:
      - transient_error
      - timeout
      - agent_crash

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
george modify job/GEORGE-123 \
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

### Modification History

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
spec:
  allowLiveModifications:
    addSteps: true
    removeSteps: pending_only
    reorderSteps: true
    modifyParameters: false  # Lock parameters after start
```

### Validation

Engine validates before applying:

```
Modification rejected:
  Cannot remove step "create-accounts" - already completed.
  Cannot reorder step "verify-access" before "provision-laptop" -
    verify-access depends on provision-laptop output.
```
