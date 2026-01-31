---
id: armory-agent-api
title: AI-Armory Server Agent API Routes
status: draft
category: armory-server
priority: high
tags: [api, fastify, typescript, prisma, socket-io, proxy]
depends-on: [wap-trust-system, wap-file-injection, wap-version-management, wap-health-monitoring]
related: [armory-agent-ui]
owner: TBD
created: 2026-01-31
updated: 2026-01-31
---

# AI-Armory Server Agent API Routes

## Summary

A new `agentControlRoutes.ts` module in the AI-Armory Server (Fastify) that proxies agent management requests to WAP. All routes require SuperTokens user authentication and verify container ownership via `ContainerLaunch.accountId`. Includes WapClient TypeScript extensions, Socket.io real-time events, and Prisma schema updates.

## Use Cases

- **As** a frontend developer, **I want** a unified REST API on the Armory Server for all agent operations, **so that** the frontend doesn't need to know about WAP's internal API structure.
- **As** a platform user, **I want** real-time Socket.io events when agent health, version, or files change, **so that** the UI updates automatically.
- **As** a security-conscious operator, **I want** all agent API calls authenticated and ownership-verified, **so that** users can only manage their own containers.

## Functional Requirements

- **FR-001**: The system SHALL expose 15 REST endpoints under `/v1/agents/` that proxy to corresponding WAP endpoints.
- **FR-002**: All endpoints SHALL require SuperTokens session authentication.
- **FR-003**: All per-container endpoints SHALL verify that the authenticated user owns the container via `ContainerLaunch.accountId`.
- **FR-004**: The WapClient class SHALL be extended with 12 new methods for health, files, version, and env operations.
- **FR-005**: The server SHALL emit 4 new Socket.io events for real-time updates.
- **FR-006**: The Prisma schema SHALL be extended with `accessParameters`, `agentCapabilities`, `instructionsDir`, and `supportsHotReload` fields on `ContainerLaunch`.

## API Contract

### REST Endpoints

All routes are prefixed with `/v1/agents/`.

| Method | Path | Description | WAP Proxy |
|--------|------|-------------|-----------|
| POST | `/:containerId/restart` | Restart container | `POST /containers/:wapId/restart` |
| POST | `/:containerId/stop` | Stop container | `POST /containers/:wapId/stop` |
| POST | `/:containerId/start` | Start container | `POST /containers/:wapId/start` |
| POST | `/:containerId/update-version` | Update version | `POST /containers/:wapId/update-image` |
| GET | `/:containerId/versions` | List versions | Local query |
| GET | `/:containerId/files` | List files | `GET /containers/:wapId/files` |
| GET | `/:containerId/files/*path` | Read file | `GET /containers/:wapId/files/read` |
| PUT | `/:containerId/files/*path` | Write file | `PUT /containers/:wapId/files/write` |
| DELETE | `/:containerId/files/*path` | Delete file | `DELETE /containers/:wapId/files` |
| GET | `/health` | All health summary | `GET /containers/health` |
| GET | `/:containerId/health` | Single health | `GET /containers/:wapId/health` |
| GET | `/:containerId/stats` | Resource stats | `GET /containers/:wapId/stats` |
| GET | `/:containerId/logs` | Recent logs | `GET /containers/:wapId/logs` |
| GET | `/:containerId/env` | Get env vars | `GET /containers/:wapId/env` |
| PATCH | `/:containerId/env` | Update env vars | `PATCH /containers/:wapId/env` |

### Socket.io Events

| Event | Direction | Data |
|-------|-----------|------|
| `agent-health-update` | Server -> Client | `{ containerId, health, resources }` |
| `agent-version-update` | Server -> Client | `{ containerId, oldTag, newTag, status }` |
| `agent-file-changed` | Server -> Client | `{ containerId, path, action }` |
| `agent-config-reloaded` | Server -> Client | `{ containerId, files }` |

## Data Model

### Prisma Schema Extension

```prisma
model ContainerLaunch {
    // ... existing fields ...

    accessParameters   Json?     // { writablePaths, readOnlyPaths, configDir, ... }
    agentCapabilities  String[]  // ["file.read", "file.write", "restart", "hot-reload"]
    instructionsDir    String?   // "/app/config/instructions"
    supportsHotReload  Boolean   @default(false)
}
```

### WapClient TypeScript Extensions

```typescript
// Health & monitoring
async getContainerHealth(containerId: string): Promise<ContainerHealth>
async getAllContainerHealth(): Promise<ContainerHealthSummary>
async getContainerStats(containerId: string): Promise<ContainerStats>
async getContainerLogs(containerId: string, options: LogOptions): Promise<string>

// File operations
async listContainerFiles(containerId: string, path: string): Promise<FileEntry[]>
async readContainerFile(containerId: string, path: string): Promise<FileContent>
async writeContainerFile(containerId: string, path: string, content: string): Promise<void>
async deleteContainerFile(containerId: string, path: string): Promise<void>

// Version management
async updateContainerImage(
    containerId: string,
    image: string,
    tag: string
): Promise<UpdateResult>

// Environment
async getContainerEnv(containerId: string): Promise<EnvVar[]>
async updateContainerEnv(
    containerId: string,
    env: Record<string, string>
): Promise<void>
```

## Implementation Notes

- The route module registers with Fastify using `fastify.register(agentControlRoutes, { prefix: '/v1/agents' })`.
- Container ownership check pattern: query `ContainerLaunch` by `containerId`, compare `accountId` to session user.
- The WapClient uses HTTP Basic Auth to authenticate with WAP Core.
- Socket.io events should be scoped to the project room so only relevant users receive updates.

`[NEEDS CLARIFICATION]` How does the server map `containerId` (Armory's ID) to `wapId` (WAP's container reference)? Is this stored in `ContainerLaunch` or resolved at request time?

`[NEEDS CLARIFICATION]` Should the `/health` summary endpoint filter containers by the authenticated user's projects, or return all containers?

## Acceptance Criteria

- **AC-001**: All 15 endpoints return correct responses when called with valid authentication and ownership.
- **AC-002**: Unauthenticated requests return 401.
- **AC-003**: Requests for containers not owned by the user return 403.
- **AC-004**: Socket.io events are emitted when agent state changes (health, version, file, config).
- **AC-005**: The Prisma migration adds all 4 new fields without data loss.
- **AC-006**: WapClient methods correctly proxy to WAP Core endpoints with Basic Auth.

## Dependencies

- [WAP Trust System](../wap-trust-system/spec.md) — WAP-side authorization.
- [WAP File Injection](../wap-file-injection/spec.md) — File operation endpoints.
- [WAP Version Management](../wap-version-management/spec.md) — Update and env endpoints.
- [WAP Health Monitoring](../wap-health-monitoring/spec.md) — Health and stats endpoints.

## Research References

- **Fastify**: Route registration, schema validation, hooks for auth
- **Prisma**: Migration workflow for schema changes
- **Socket.io**: Room-based event emission for scoped updates

## Out of Scope

- Rate limiting on agent control endpoints
- Batch operations (e.g., restart all agents in a project)
- WebSocket-based log streaming (SSE proxied from WAP is sufficient)
