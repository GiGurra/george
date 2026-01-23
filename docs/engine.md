# George Engine

The George engine is the core reconciliation service that watches for jobs and manages agent lifecycles.

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Jira (Storage + Visualization)           │
│         Jobs, steps, status, comments, evidence, audit      │
└─────────────────────────────┬───────────────────────────────┘
                              │ watches / updates
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    George Engine (Go service)               │
│    - Watches Jira for new jobs / state changes              │
│    - Spawns agents when work is needed                      │
│    - Monitors agent health                                  │
│    - Restarts failed agents                                 │
│    - Terminates completed agents                            │
│    - Writes status back to Jira                             │
└─────────────────────────────┬───────────────────────────────┘
                              │ spawns / manages
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Agents (Containers)                      │
│              Spawned via Docker or Kubernetes               │
└─────────────────────────────────────────────────────────────┘
```

## Implementation

The engine is a simple Go application that:

1. **Watches** Jira for new/changed jobs (polling or webhooks)
2. **Reconciles** desired state with actual state
3. **Spawns** agents as containers (Docker or K8s pods)
4. **Monitors** agent health and job progress
5. **Updates** Jira with status changes

### Reconciliation Loop

```go
// Pseudocode
for {
    jobs := fetchJobsFromJira()

    for _, job := range jobs {
        switch job.Status {
        case "pending":
            if agentAvailable(job) {
                spawnAgent(job)
                updateJobStatus(job, "in_progress")
            }
        case "in_progress":
            agent := getAgentForJob(job)
            if agent == nil || !agent.IsHealthy() {
                // Agent died, respawn
                spawnAgent(job)
            }
        case "blocked":
            // Check if block condition resolved
            if canUnblock(job) {
                spawnAgent(job)
                updateJobStatus(job, "in_progress")
            }
        case "completed", "failed":
            // Cleanup
            terminateAgent(job)
        }
    }

    sleep(reconcileInterval)
}
```

### Container Backends

The engine supports multiple container runtimes:

**Docker:**
```go
type DockerBackend struct {
    client *docker.Client
}

func (d *DockerBackend) SpawnAgent(job Job, spec AgentSpec) (string, error) {
    return d.client.ContainerCreate(ctx, &container.Config{
        Image: spec.Image,
        Env: []string{
            "GEORGE_JOB_ID=" + job.ID,
            "GEORGE_JIRA_URL=" + config.JiraURL,
            // Capabilities passed as env or mounted secrets
        },
    }, nil, nil, nil, "")
}
```

**Kubernetes:**
```go
type K8sBackend struct {
    client *kubernetes.Clientset
}

func (k *K8sBackend) SpawnAgent(job Job, spec AgentSpec) (string, error) {
    pod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name: fmt.Sprintf("george-agent-%s", job.ID),
            Labels: map[string]string{
                "george.io/job": job.ID,
                "george.io/template": job.Template,
            },
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{{
                Name:  "agent",
                Image: spec.Image,
                Env:   buildEnvVars(job, spec),
            }},
            RestartPolicy: corev1.RestartPolicyNever,
        },
    }
    return k.client.CoreV1().Pods(namespace).Create(ctx, pod, metav1.CreateOptions{})
}
```

## Agent Lifecycle

### Spawn

1. Engine detects job needs processing (new job, or needs retry)
2. Engine selects suitable AgentSpec based on `requiredCapabilities`
3. Engine spawns agent container with:
   - Job reference (Jira issue key)
   - Capability credentials (env vars or mounted secrets)
   - Engine API endpoint (for status updates)

### Execute

1. Agent starts, reads job spec from Jira
2. Agent reads conversation history from Jira comments
3. Agent determines current state: "where am I in this job?"
4. Agent decides next action based on LLM interpretation
5. Agent executes (idempotently)
6. Agent updates Jira (status, comments, evidence)
7. Repeat until done or blocked

### Terminate

- **Completed**: Agent marks job as completed, container exits cleanly
- **Blocked**: Agent marks job as blocked with reason, container exits
- **Crash**: Engine detects container died, respawns with same job context

## Engine Configuration

```yaml
# george-config.yaml
engine:
  reconcileInterval: 30s

jira:
  url: https://company.atlassian.net
  # Service account credentials
  credentials:
    secretRef: george-jira-credentials
  project: GEORGE

containerBackend:
  type: kubernetes  # or "docker"
  kubernetes:
    namespace: george-agents
    serviceAccount: george-agent
  docker:
    network: george-network

agents:
  # Default resource limits for agent containers
  defaultResources:
    memory: 512Mi
    cpu: 500m

  # Health check settings
  healthCheck:
    interval: 10s
    timeout: 5s
    unhealthyThreshold: 3
```

## Engine API

The engine exposes an API for the CLI and agents:

```
GET  /api/v1/jobs                    # List all jobs
GET  /api/v1/jobs/{id}               # Get job details
POST /api/v1/jobs/{id}/pause         # Pause job
POST /api/v1/jobs/{id}/resume        # Resume job
POST /api/v1/jobs/{id}/abort         # Abort job

GET  /api/v1/agents                  # List running agents
GET  /api/v1/agents/{id}             # Get agent details
GET  /api/v1/agents/{id}/logs        # Stream agent logs

POST /api/v1/jobs/{id}/override      # Manual override
POST /api/v1/jobs/{id}/inject        # Inject message to agent

WS   /api/v1/jobs/{id}/watch         # WebSocket for real-time updates
```

## Scaling Considerations

For v1, a single engine instance is fine. For scale:

- **Multiple engines**: Use leader election (lease in K8s, or external lock)
- **Job sharding**: Partition jobs by project/team
- **Agent pools**: Pre-warm agent containers for faster startup

## Deployment

### Docker Compose (Development)

```yaml
version: '3.8'
services:
  george-engine:
    image: george/engine:latest
    environment:
      - GEORGE_JIRA_URL=https://company.atlassian.net
      - GEORGE_JIRA_TOKEN=${JIRA_TOKEN}
      - GEORGE_CONTAINER_BACKEND=docker
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "8080:8080"
```

### Kubernetes (Production)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: george-engine
spec:
  replicas: 1  # Single instance with leader election, or shard
  selector:
    matchLabels:
      app: george-engine
  template:
    metadata:
      labels:
        app: george-engine
    spec:
      serviceAccountName: george-engine
      containers:
        - name: engine
          image: george/engine:latest
          env:
            - name: GEORGE_JIRA_URL
              valueFrom:
                secretKeyRef:
                  name: george-secrets
                  key: jira-url
          ports:
            - containerPort: 8080
```
