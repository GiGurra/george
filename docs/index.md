# George

A GitOps-driven LLM agent reconciliation engine.

## What is this?

George is a platform for orchestrating LLM agents using GitOps principles and Kubernetes-style reconciliation. The core idea:

1. **Define** agent capabilities, task templates, and job instances declaratively in Git
2. **Reconcile** desired state (Git) with actual state (running agents, task progress)
3. **Observe** everything through a dashboard - steps, sub-tasks, human handoffs, completions
4. **Resume** seamlessly - agents are stateless and reconstruct their context from external state

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

## Key Principles

- **Git as source of truth** - All definitions live in version control
- **Reconciliation loop** - Kubernetes-style continuous reconciliation
- **Stateless agents** - Agents reconstruct context from external state (CRDs + DB)
- **Idempotent operations** - Steps can be safely retried
- **Human-in-the-loop** - First-class support for human tasks and approvals
- **Observable** - Full visibility into all steps, sub-tasks, and state transitions

## Documentation

- [Design Discussion](design-discussion.md) - How we arrived at this design
- [Architecture](architecture.md) - Technical details and resource schemas
- [Naming](naming.md) - Project name alternatives (George is a working title)
