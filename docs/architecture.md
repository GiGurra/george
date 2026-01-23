# Architecture

High-level architecture of George - an LLM agent reconciliation engine.

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         User                                │
│            Creates issue in Jira, fills form                │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Jira (Templates + Jobs + Status)               │
│                                                             │
│  Issue Types = Task Templates                               │
│  Custom Fields = Parameters                                 │
│  Issues = Jobs                                              │
│  Subtasks = Steps                                           │
│  Comments = Agent activity log                              │
└─────────────────────────────┬───────────────────────────────┘
                              │ watches / updates
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    George Engine (Go service)               │
│                                                             │
│  ┌─────────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │  Reconciliation │  │   Scheduler │  │      API        │ │
│  │      Loop       │  │             │  │   (CLI, MCP)    │ │
│  └────────┬────────┘  └──────┬──────┘  └─────────────────┘ │
└───────────┼──────────────────┼──────────────────────────────┘
            │                  │
            │ spawns/manages   │
            ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│                    Agents (Containers)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Agent 1   │  │   Agent 2   │  │   Agent N   │         │
│  │ (job: 123)  │  │ (job: 456)  │  │ (job: ...)  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│              Spawned via Docker or Kubernetes               │
└─────────────────────────────────────────────────────────────┘
            │
            │ interacts with
            ▼
┌─────────────────────────────────────────────────────────────┐
│                    External Services                        │
│         Slack, GitHub, Calendars, APIs, Humans              │
└─────────────────────────────────────────────────────────────┘
```

## V1: Jira-Native (No GitOps)

For v1, everything lives in Jira:

| Concept | Jira Implementation |
|---------|---------------------|
| Task Template | Issue Type + Custom Fields + Create Screen |
| Template Parameters | Required Custom Fields |
| Steps | Subtasks or Checklist |
| Job Instance | Issue of that type |
| Job Status | Issue Status + Subtask statuses |
| Agent Activity | Comments |
| Evidence | Attachments + Comments |

**Benefits:**
- No git sync needed
- Users stay in familiar Jira UI
- Faster to prototype
- One less integration point

**GitOps can come later** if we need version-controlled templates, PR review for changes, or cross-environment replication.

## Components

| Component | Description | Details |
|-----------|-------------|---------|
| **Jira** | Primary storage and visualization | [Integrations](integrations.md#jira-primary-storage--visualization) |
| **George Engine** | Go service running reconciliation | [Engine](engine.md) |
| **Agents** | Containerized workers executing jobs | [Engine - Agent Lifecycle](engine.md#agent-lifecycle) |
| **CLI** | Command-line interface | [Integrations](integrations.md#cli-george) |
| **MCP** | AI assistant integration | [Integrations](integrations.md#mcp--claude-code-skills) |

## Core Concepts

| Concept | Description | Details |
|---------|-------------|---------|
| **AgentSpec** | Defines agent capabilities and constraints | [Resources](resources.md#agentspec) |
| **TaskTemplate** | Defines a type of work (textual, interpreted by agent) | [Resources](resources.md#tasktemplate) |
| **Job** | Instance of a template with parameters | [Resources](resources.md#job) |
| **Step** | Individual unit of work within a job | [Resources](resources.md#status-states) |

## Key Principles

1. **Jira-native** - Jira is the interface for users, storage for state
2. **Reconciliation** - Engine continuously reconciles desired vs actual state
3. **Stateless agents** - Agents reconstruct context from Jira
4. **Idempotent operations** - Steps can be safely retried
5. **Human-in-the-loop** - First-class support for human tasks
6. **Observable** - Full visibility in tools you already use

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
- Jira comments for all agent actions
- Job modifications tracked with timestamps
- Agent container logs for debugging

## Future Considerations

### Template Versioning

What happens when a template changes while jobs are running?

- **Pin on start** (likely) - Job uses template version from when it started
- **Live update** - Job picks up template changes (risky)
- **Manual migration** - Operator decides per-job

### Rollback / Compensation

If step 3 fails, should we undo steps 1-2?

```yaml
steps:
  - name: create-accounts
    compensate:
      description: "Delete created accounts"
```

Complex - not all actions are reversible. Probably a v2 feature.

### Concurrency Control

What if two jobs need the same resource?

```yaml
spec:
  locks:
    - resource: "slack:#it-requests"
      scope: exclusive
```

### Cost Tracking

Track LLM token usage per job:

```yaml
status:
  costs:
    totalTokens: 45000
    estimatedCost: "$0.45"
```

### Agent-to-Agent Communication

Jobs spawning sub-jobs handled by different agents:

```yaml
steps:
  - name: request-security-review
    delegate_to:
      template: security-review
      wait: true
```

### Recursive Reconciliation

Agents spawning other tracked agents. The engine would manage:
- Parent-child job relationships
- Cascading cancellation
- Aggregated status views

### GitOps Integration

If needed later, templates could be managed in Git:

- Template definitions in YAML, synced to Jira issue types
- PR review before template changes go live
- Version history of template evolution
- Cross-environment replication

For v1, Jira-native is simpler and sufficient.

## Documentation Map

- [Design Discussion](design-discussion.md) - Why we made these choices
- [Resources](resources.md) - AgentSpec, TaskTemplate, Job schemas
- [Engine](engine.md) - George engine implementation
- [Integrations](integrations.md) - Jira, Slack, CLI, MCP
- [Operations](operations.md) - Human intervention, debugging, live modifications
- [Naming](naming.md) - Project name alternatives
