# Design Discussion

This document captures the reasoning and discussion that led to George's design. Understanding *why* decisions were made is as important as understanding *what* was decided.

## The Problem

Organizations have many manual processes that:

- Take significant calendar time waiting for approvals and work from different teams
- Are tedious, repetitive, and time-consuming
- Take a long time to fully automate with traditional tooling
- Are hard to track, resume, and get an overview of

LLM agents could perform these tasks, but managing agents at scale introduces new challenges:

- Tracking what's been ordered, started, completed
- Resuming stopped or crashed agents
- Visualizing progress and status
- Maintaining audit trails

## The Core Insight: GitOps + Reconciliation

What if we applied GitOps and Kubernetes-style reconciliation to agent orchestration?

1. **Define agent capabilities** (not related to a particular flow/task) - in Git
2. **Define task templates** with variables (fuzzy or strict) - in Git
3. **Commit job instances** (template + parameters) - in Git
4. **Reconciliation engine** watches Git and ensures actual state matches desired state
5. **Agents** execute the work, reporting status back to the engine
6. **Dashboard** visualizes everything - available resources, jobs, steps, sub-tasks

## Key Design Decisions

### Why Not Temporal.io?

Temporal is excellent for durable workflow execution, but it assumes:

- **Deterministic replay** - same inputs produce same execution path
- **Workflows as code** - the workflow IS the program

Our requirements are different:

- **Non-deterministic execution** - LLMs interpret tasks at runtime
- **Workflows as text** - natural language task descriptions interpreted by agents
- **Fuzzy steps** - "wait for IT to confirm in Slack" not `workflow.ExecuteActivity(WaitForConfirmation)`

Temporal's replay mechanism would fight against LLM non-determinism rather than help.

### Stateless Agents with External State

Instead of agents maintaining internal state, we externalize everything:

- **CRDs** hold job/task/step structure and status metadata
- **Database** holds conversation context and detailed history
- **Agents are ephemeral** - when they die, a new one spawns, reads state, continues

This gives us:

- **Crash recovery** = spawn new agent, it re-reads everything
- **Upgrades** = deploy new agent version, it interprets same state (possibly better)
- **Debugging** = state is all in CRD + DB, fully inspectable
- **Auditability** = context history is explicit, not hidden in runtime

### Engine vs Agent Responsibilities

Clear separation of concerns:

| Engine | Agent |
|--------|-------|
| Knows structure (steps, order, dependencies) | Knows semantics (what work means) |
| Tracks status transitions | Determines if done_when is satisfied |
| Visualizes progress | Executes the actual work |
| Spawns/manages agent pods | Uses tools, talks to humans |
| Reacts to status (alerts, retries) | Reports success/failure/blocked |
| Doesn't interpret task content | Introspects its own capabilities |

The engine understands the flow well enough to visualize it, but doesn't understand the details of the steps. Only the agent understands the actual work.

### Mixed Execution Modes

Not everything needs an LLM. The agent decides how to execute based on the task:

```
Simple                                              Complex
  │                                                    │
  ▼                                                    ▼
CLI return code    API response     Slack ack     LLM judgment
```

A step might be as simple as checking a CLI return code, or as complex as coordinating with humans in Slack.

### Capability-Aware Agents

Agents should know what tools they have. If they can't do a task:

- **Don't hack around it** with wild workarounds
- **Stop and report** that the task cannot be performed
- **Set clear status** - capability missing, conditions not met, etc.

This prevents agents from doing damage when they lack proper access.

### Idempotent Operations

Steps should be idempotent - safe to retry. This enables:

- Retry with backoff (like pods in a deployment)
- Resume after crash
- Multiple agents potentially working on same job (with coordination)

## Alternatives Considered

### Kubernetes as Reconciliation Engine

**Pros:**
- Battle-tested reconciliation
- CRDs are flexible
- Huge ecosystem
- Status subresources for observability

**Cons:**
- Heavy infrastructure
- YAML verbosity
- Not designed for long-running workflows (pod eviction)

**Verdict:** Good fit for control plane, but agents need careful design around pod lifecycle.

### Crossplane

Crossplane adds composition abstractions on top of Kubernetes, but it's designed for infrastructure provisioning (databases, cloud resources), not workflow orchestration.

**Verdict:** Wrong abstraction layer. We'd still write custom controllers anyway.

### Custom Go Service

A simple approach:
1. Watch git repo (webhook or polling)
2. Store state in Postgres
3. Run reconciliation loop
4. Spawn containerized agents

**Verdict:** Viable for prototyping. Less infrastructure overhead. Can graduate to K8s later.

### Hybrid Architecture

Separate concerns:

```
┌─────────────────────────────────────────────────────┐
│  Git (Source of Truth)                              │
│  - agent-specs/                                     │
│  - task-templates/                                  │
│  - jobs/ (instances)                                │
└─────────────────┬───────────────────────────────────┘
                  │ GitOps sync
                  ▼
┌─────────────────────────────────────────────────────┐
│  Control Plane (K8s CRDs or lightweight DB)         │
│  - AgentSpec, TaskTemplate, Job resources           │
│  - Owns "what should exist"                         │
└─────────────────┬───────────────────────────────────┘
                  │ reconciliation
                  ▼
┌─────────────────────────────────────────────────────┐
│  Execution Engine                                   │
│  - Actually runs the workflows                      │
│  - Handles durability, retry, human-wait            │
│  - Reports status back to control plane             │
└─────────────────────────────────────────────────────┘
```

## Open Questions

1. **State persistence details** - Exactly how to partition between CRDs (metadata) and DB (large context)?
2. **Agent runtime** - Containerized agents vs shared agent pool?
3. **Recursive reconciliation** - Agents spawning sub-agents tracked as CRDs?
4. **Multi-tenancy** - Multiple teams/projects sharing one George instance?
5. **Security model** - How to safely scope agent capabilities?

## Future Considerations

- Recursive reconciliation (agents spawning tracked sub-agents)
- Agent-to-agent communication
- Learning from completed tasks to improve future execution
- Cost tracking and optimization
