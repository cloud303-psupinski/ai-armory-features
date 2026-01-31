---
id: wap-health-monitoring
title: WAP Enhanced Health Monitoring
status: draft
category: wap-proxy
priority: high
tags: [health, monitoring, stats, logs, sse, nats]
depends-on: [wap-trust-system]
related: [armory-agent-api, armory-agent-ui]
owner: TBD
created: 2026-01-31
updated: 2026-01-31
---

# WAP Enhanced Health Monitoring

## Summary

Aggregated health, resource usage (CPU/memory/network/disk), and log streaming endpoints for all managed containers. Provides both snapshot queries and real-time SSE streams. Data flows from Docker Engine through Bollard, over NATS, to WAP Core HTTP endpoints.

## Use Cases

- **As** a platform operator, **I want** a single endpoint that returns the health status of all agent containers, **so that** I can build a dashboard showing overall system health.
- **As** a developer, **I want** to stream container logs in real time, **so that** I can debug agent behavior without SSH access to the host.
- **As** the AI-Armory Frontend, **I want** CPU and memory usage per container, **so that** I can display resource gauges in the agent management UI.

## Functional Requirements

- **FR-001**: The system SHALL provide a health summary endpoint returning status, health, and resource metrics for all containers.
- **FR-002**: The system SHALL provide a per-container detailed health endpoint.
- **FR-003**: The system SHALL provide a per-container resource stats endpoint with CPU%, memory usage/limit, network I/O, block I/O, and PID count.
- **FR-004**: The system SHALL provide a log streaming endpoint using Server-Sent Events (SSE) with `tail` and `follow` parameters.
- **FR-005**: Health data SHALL flow through NATS subjects from the Rust agent to the Go core.
- **FR-006**: Each container's health response SHALL include Docker labels for agent type and project identification.

## API Contract

### NATS Subjects

| Subject | Description |
|---------|-------------|
| `docker.container.health` | Single container health status |
| `docker.container.health.all` | All containers summary |
| `docker.container.stats` | Resource usage stats |
| `docker.container.stats.stream` | Streaming stats |

### HTTP Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/containers/health` | All containers health summary |
| GET | `/api/v1/containers/:id/health` | Single container detailed health |
| GET | `/api/v1/containers/:id/stats` | CPU/memory/network stats |
| GET | `/api/v1/containers/:id/logs?tail=100&follow=true` | Container logs (SSE) |

### Health Summary — Response

```json
{
    "timestamp": "2026-01-31T10:45:00Z",
    "total": 5,
    "healthy": 4,
    "unhealthy": 1,
    "containers": [
        {
            "id": "armory-ideation",
            "name": "armory-ideation",
            "image": "ai-armory-client:v1.0.0",
            "status": "running",
            "health": "healthy",
            "uptime": "2h30m15s",
            "resources": {
                "cpuPercent": 12.5,
                "memoryUsageMB": 245,
                "memoryLimitMB": 8192,
                "memoryPercent": 2.99,
                "networkRxMB": 1.02,
                "networkTxMB": 0.51,
                "blockReadMB": 15.3,
                "blockWriteMB": 2.1,
                "pids": 12
            },
            "labels": {
                "forge.agent.type": "ideation",
                "forge.project.id": "cml0eisc80001p214tg6z7v5t"
            },
            "lastHealthCheck": "2026-01-31T10:44:55Z"
        }
    ]
}
```

### Log Streaming — SSE Format

```
GET /api/v1/containers/armory-ideation/logs?tail=50&follow=true

Content-Type: text/event-stream

event: log
data: {"timestamp":"2026-01-31T10:44:55Z","stream":"stdout","message":"[INFO] Message received in room cml0eixyh"}

event: log
data: {"timestamp":"2026-01-31T10:44:56Z","stream":"stdout","message":"[INFO] Invoking Claude Code CLI..."}
```

## Data Model

No additional database tables. Health and stats data is read in real time from Docker Engine via Bollard. The `labels` field in the response maps to Docker container labels set at creation time.

## Implementation Notes

- The Rust agent uses `docker.stats(container_id, stream: false)` for snapshot stats and `stream: true` for the streaming variant.
- Health status is derived from Docker's built-in `HEALTHCHECK` if configured, otherwise inferred from container state.
- Log streaming uses `docker.logs(container_id, follow: true, tail: N)` piped through SSE.
- The summary endpoint should be efficient — batch-query all containers rather than N+1 individual calls.

`[NEEDS CLARIFICATION]` Should the health endpoint require trust checks, or should it be open to any authenticated service? Currently the per-container stats endpoint has no explicit trust check listed.

## Acceptance Criteria

- **AC-001**: The health summary endpoint returns data for all managed containers in a single response.
- **AC-002**: Each container entry includes `status`, `health`, `uptime`, and `resources` fields.
- **AC-003**: Resource stats include `cpuPercent`, `memoryUsageMB`, `memoryLimitMB`, `networkRxMB`, `networkTxMB`.
- **AC-004**: The logs endpoint streams new log lines in real time when `follow=true`.
- **AC-005**: The logs endpoint returns only the last N lines when `tail=N` is specified.
- **AC-006**: SSE log events include `timestamp`, `stream` (stdout/stderr), and `message` fields.

## Dependencies

- [WAP Trust System](../wap-trust-system/spec.md) — Trust checks for per-container access (scope TBD).

## Research References

- **cAdvisor**: Container resource monitoring and performance analysis
- **Docker Stats API**: `/containers/{id}/stats` endpoint for resource metrics
- **Prometheus**: Standard metrics format for container monitoring exporters

## Out of Scope

- Historical metrics storage or time-series database integration
- Alerting rules or notification triggers
- Custom health check definitions (relies on Docker HEALTHCHECK)
- Metrics aggregation across multiple Docker hosts
