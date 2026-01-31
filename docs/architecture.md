---
id: architecture-overview
title: AI-Armory Platform Architecture
status: stable
updated: 2026-01-31
---

# AI-Armory Platform Architecture

## Executive Summary

Two interconnected features that enable the AI-Armory frontend to manage AI agent containers through the WAP (Web Application Platform) Docker Manager. Includes trust-based access control, file injection into running containers, real-time health monitoring, version management, and environment configuration.

## Components

| Component | Stack | Role |
|-----------|-------|------|
| **WAP Docker Manager** | Go 1.23 (Gin) + Rust (Bollard/Tokio) + NATS 2.10 + SQLite | Docker container management via NATS message bus to Rust agent that communicates with Docker socket via Bollard SDK. SSE real-time events. Template/blueprint deployment system. Hierarchical configuration with materialized path pattern. |
| **AI-Armory Server** | TypeScript (Fastify 5) + Prisma + PostgreSQL + Redis + Socket.io | Orchestration layer for AI agents. Manages projects, rooms, sessions, containers. Calls WAP HTTP API (Basic Auth) for container lifecycle. Syncs container status via polling (30s) + SSE events. |
| **AI-Armory Frontend** | Next.js 16 + Tailwind CSS | User-facing UI for projects, rooms, chat with AI agents. Real-time updates via Socket.io. |
| **Agent Containers** | TypeScript + Claude Code CLI + Memvid memory | Long-lived Docker containers running Claude Code. Connect to server via WebSocket. Per-room chat sessions with conversation history. Semantic memory via memvid SDK. |

## Current Integration Flow

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

## What Exists

- Container create/start/stop/restart/remove via WAP API
- Container status sync (30s polling + SSE events)
- WAP template caching in AI-Armory server
- Machine token auth for agent containers
- WebSocket real-time events for container status changes
- Docker events streaming via SSE

## What's Missing

- No trust relationship model between services/containers. See [WAP Trust System](specs/wap-trust-system/spec.md).
- No file injection/update capability for running containers. See [WAP File Injection](specs/wap-file-injection/spec.md).
- No agent instruction file management (CLAUDE.md editing). See [WAP File Injection](specs/wap-file-injection/spec.md).
- No version management (rolling updates). See [WAP Version Management](specs/wap-version-management/spec.md).
- No real-time health aggregation dashboard. See [WAP Health Monitoring](specs/wap-health-monitoring/spec.md).
- No parameterized access control (per-path, per-action). See [WAP Trust System](specs/wap-trust-system/spec.md).
- No environment variable management UI. See [Armory Agent UI](specs/armory-agent-ui/spec.md).
- No container logs viewer. See [WAP Health Monitoring](specs/wap-health-monitoring/spec.md).

## Infrastructure

| Service | Purpose |
|---------|---------|
| Traefik | Reverse proxy, TLS termination, routing |
| PostgreSQL | AI-Armory Server database |
| Redis | Cache, Socket.io adapter |
| MinIO | S3-compatible object storage |
| NATS | WAP internal message bus |
| SuperTokens | Authentication (OAuth, email/password) |
