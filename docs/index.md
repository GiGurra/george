# George

An LLM agent reconciliation engine with Jira as the primary interface.

## What is this?

George is a platform for orchestrating LLM agents using Kubernetes-style reconciliation. The core idea:

1. **Order work** via Jira - create an issue, fill in the form, done
2. **Reconcile** - engine watches Jira and spawns agents to do the work
3. **Observe** everything in Jira - steps, sub-tasks, human handoffs, completions
4. **Resume** seamlessly - agents are stateless and reconstruct context from Jira

## Why?

Enterprise environments often have manual processes that:

- Take calendar time (days/weeks) waiting for approvals and work from different teams
- Are tedious and repetitive
- Are hard to track and audit
- Are difficult to fully automate with traditional tooling

LLM agents can help, but managing them at scale introduces new problems:

- How do you track what's been ordered, started, completed?
- How do you resume if an agent crashes?
- How do you visualize progress across many concurrent jobs?
- How do you audit what happened?

George applies GitOps principles to solve these problems.

## Key Components

| Component | Description |
|-----------|-------------|
| **Jira** | Primary storage and visualization layer |
| **George Engine** | Go service that runs the reconciliation loop |
| **Agents** | Containerized workers (Docker or K8s) |
| **CLI** | `george` command-line tool |
| **MCP** | Integration with Claude Code and AI assistants |

## Key Principles

- **Jira-native** - Use Jira for everything: templates, jobs, status, history
- **Reconciliation** - Kubernetes-style continuous reconciliation
- **Stateless agents** - Agents reconstruct context from Jira
- **Idempotent operations** - Steps can be safely retried
- **Human-in-the-loop** - First-class support for human tasks and approvals
- **Observable** - Full visibility in a tool you already use

## Documentation

- [Design Discussion](design-discussion.md) - How we arrived at this design, trade-offs considered
- [Architecture](architecture.md) - High-level system architecture
- [Resources](resources.md) - AgentSpec, TaskTemplate, Job schemas
- [Engine](engine.md) - George engine implementation details
- [Integrations](integrations.md) - Jira, Slack, CLI, MCP
- [Operations](operations.md) - Human intervention, debugging, live modifications
- [Naming](naming.md) - Project name alternatives (George is a working title)

## Quick Example

```yaml
# TaskTemplate: Define what work looks like
apiVersion: george.io/v1
kind: TaskTemplate
metadata:
  name: employee-onboarding
spec:
  description: "Onboard a new employee"
  parameters:
    - name: employeeName
      type: string
  steps:
    - name: create-accounts
      description: "Create accounts in all required systems"
    - name: provision-laptop
      description: "Request IT to provision laptop"
    - name: verify-access
      description: "Verify employee can log into all systems"
```

```bash
# Create a job
george run template/employee-onboarding \
  --param employeeName="Alice Smith"

# Watch progress
george attach job/GEORGE-123
```

## Status

Early concept/design phase. See the documentation for design discussions and architecture.
