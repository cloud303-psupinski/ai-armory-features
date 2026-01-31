---
id: architecture-overview
title: AI-Armory Platform Architecture
status: stable
updated: 2026-01-31
---

# AI-Armory Platform Architecture

## Component Overview

| Component | Stack | Role |
|-----------|-------|------|
| **WAP Docker Manager** | Go 1.23 (Gin) + Rust (Bollard/Tokio) + NATS 2.10 + SQLite | Container orchestration and trust-based access control |
| **AI-Armory Server** | TypeScript (Fastify 5) + Prisma + PostgreSQL + Redis + Socket.io | Application backend, user auth, project management |
| **AI-Armory Frontend** | Next.js 16 + Tailwind CSS | User interface for agent and project management |
| **Agent Containers** | TypeScript + Claude Code CLI + Memvid memory | AI agent runtime environments |

## Integration Flow

```
AI-Armory Frontend (Next.js)
    | Socket.io / REST
    v
AI-Armory Server (Fastify)
    | HTTP Basic Auth
    v
WAP Core (Go/Gin)
    | NATS messages
    v
WAP Agent (Rust/Bollard)
    | Docker socket
    v
Docker Engine
```

## Current Capabilities

- Container lifecycle management (create/start/stop/restart/remove) via WAP API
- Container status synchronization (30s polling + SSE events)
- WAP template caching for container configuration
- Machine token authentication between services
- WebSocket real-time events for frontend updates
- Docker events streaming via Server-Sent Events

## Gaps Addressed by Feature Specs

- **Trust system** — No inter-service permission model exists. See [WAP Trust System](specs/wap-trust-system/spec.md).
- **File injection** — No way to read/write files inside running containers. See [WAP File Injection](specs/wap-file-injection/spec.md).
- **Version management** — No rolling-update or env-var management. See [WAP Version Management](specs/wap-version-management/spec.md).
- **Health monitoring** — No aggregated health or resource metrics. See [WAP Health Monitoring](specs/wap-health-monitoring/spec.md).
- **Server API** — No Armory Server routes for agent control. See [Armory Agent API](specs/armory-agent-api/spec.md).
- **Frontend UI** — No agent management dashboard or file editor. See [Armory Agent UI](specs/armory-agent-ui/spec.md).
- **Config watcher** — No hot-reload for agent instructions. See [Agent Config Watcher](specs/agent-config-watcher/spec.md).

## Infrastructure

| Service | Purpose |
|---------|---------|
| Traefik | Reverse proxy, TLS termination |
| PostgreSQL | AI-Armory Server database |
| Redis | Cache, Socket.io adapter |
| MinIO | S3-compatible object storage |
| NATS | WAP internal message bus |
| SuperTokens | User authentication |
