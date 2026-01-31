# AI-Armory Platform Feature Specification

## 1. Executive Summary

Two interconnected features that enable the AI-Armory frontend to manage AI agent containers through the WAP (Web Application Platform) Docker Manager. Includes trust-based access control, file injection into running containers, real-time health monitoring, version management, and environment configuration.

---

## 2. Current Architecture

### Components

| Component | Stack | Role |
|-----------|-------|------|
| **WAP Docker Manager** | Go 1.23 (Gin) + Rust (Bollard/Tokio) + NATS 2.10 + SQLite | Docker container management via NATS message bus to Rust agent that communicates with Docker socket via Bollard SDK. SSE real-time events. Template/blueprint deployment system. Hierarchical configuration with materialized path pattern. |
| **AI-Armory Server** | TypeScript (Fastify 5) + Prisma + PostgreSQL + Redis + Socket.io | Orchestration layer for AI agents. Manages projects, rooms, sessions, containers. Calls WAP HTTP API (Basic Auth) for container lifecycle. Syncs container status via polling (30s) + SSE events. |
| **AI-Armory Frontend** | Next.js 16 + Tailwind CSS | User-facing UI for projects, rooms, chat with AI agents. Real-time updates via Socket.io. |
| **Agent Containers** | TypeScript + Claude Code CLI + Memvid memory | Long-lived Docker containers running Claude Code. Connect to server via WebSocket. Per-room chat sessions with conversation history. Semantic memory via memvid SDK. |

### Current Integration Flow

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

### What Exists

- Container create/start/stop/restart/remove via WAP API
- Container status sync (30s polling + SSE events)
- WAP template caching in AI-Armory server
- Machine token auth for agent containers
- WebSocket real-time events for container status changes
- Docker events streaming via SSE

### What's Missing

- No trust relationship model between services/containers
- No file injection/update capability for running containers
- No agent instruction file management (CLAUDE.md editing)
- No version management (rolling updates)
- No real-time health aggregation dashboard
- No parameterized access control (per-path, per-action)
- No environment variable management UI
- No container logs viewer

---

## 3. Feature 1: WAP Control Proxy

### 3.1 Trust Relationship Table

#### Purpose

Define per-service, per-container access permissions. Every operation proxied through WAP is checked against this table before execution. All operations are audit-logged.

#### Database Migration (SQLite)

```sql
-- Migration: 004_trust_relationships.up.sql

CREATE TABLE trust_relationships (
    id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    -- Identity
    caller_service TEXT NOT NULL,        -- e.g., "armory-server", "monitoring-svc"
    target_container TEXT NOT NULL,       -- e.g., "armory-ideation", "*" (wildcard)
    -- Container lifecycle permissions
    can_inspect BOOLEAN DEFAULT 1,       -- docker inspect
    can_start BOOLEAN DEFAULT 0,         -- docker start
    can_stop BOOLEAN DEFAULT 0,          -- docker stop
    can_restart BOOLEAN DEFAULT 0,       -- docker restart
    can_remove BOOLEAN DEFAULT 0,        -- docker rm
    can_exec BOOLEAN DEFAULT 0,          -- docker exec
    can_logs BOOLEAN DEFAULT 1,          -- docker logs
    can_stats BOOLEAN DEFAULT 1,         -- docker stats
    -- File access (JSON arrays of glob patterns)
    file_read_paths TEXT DEFAULT '[]',   -- e.g., ["/app/config/*", "/data/memory/*"]
    file_write_paths TEXT DEFAULT '[]',  -- e.g., ["/app/config/instructions/*"]
    -- Container update permissions
    can_update_env BOOLEAN DEFAULT 0,    -- modify environment variables
    can_update_image BOOLEAN DEFAULT 0,  -- rolling image update
    allowed_images TEXT DEFAULT '[]',    -- e.g., ["ai-armory-client:*", "armory-agent:v2.*"]
    -- Metadata
    description TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    expires_at DATETIME,                 -- NULL = never expires
    UNIQUE(caller_service, target_container)
);

CREATE TABLE trust_audit_log (
    id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    trust_id TEXT REFERENCES trust_relationships(id) ON DELETE SET NULL,
    caller_service TEXT NOT NULL,
    target_container TEXT NOT NULL,
    action TEXT NOT NULL,                -- e.g., "container.restart", "file.write"
    resource_path TEXT,                  -- file path or specific resource
    allowed BOOLEAN NOT NULL,
    reason TEXT,                          -- why allowed/denied
    request_metadata TEXT,               -- JSON: IP, user-agent, etc.
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_trust_caller ON trust_relationships(caller_service);
CREATE INDEX idx_trust_target ON trust_relationships(target_container);
CREATE INDEX idx_audit_caller ON trust_audit_log(caller_service);
CREATE INDEX idx_audit_created ON trust_audit_log(created_at);
```

#### Trust API Endpoints (WAP Core)

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/trust` | List all trust relationships | Service Auth |
| `POST` | `/api/v1/trust` | Create trust relationship | Service Auth |
| `GET` | `/api/v1/trust/:id` | Get specific relationship | Service Auth |
| `PATCH` | `/api/v1/trust/:id` | Update permissions | Service Auth |
| `DELETE` | `/api/v1/trust/:id` | Revoke trust | Service Auth |
| `GET` | `/api/v1/trust/audit` | Query audit log (filterable) | Service Auth |
| `POST` | `/api/v1/trust/check` | Check if action is allowed | Service Auth |

#### Trust Check Request/Response

```json
// POST /api/v1/trust/check
// Request
{
    "caller": "armory-server",
    "target": "armory-ideation",
    "action": "file.write",
    "resourcePath": "/app/config/instructions/CLAUDE.md"
}

// Response
{
    "allowed": true,
    "trustId": "abc123",
    "reason": "Path matches glob pattern: /app/config/instructions/*",
    "auditId": "def456"
}
```

#### Trust Check Middleware (Go)

```go
func TrustCheckMiddleware(repo *repository.TrustRepository) gin.HandlerFunc {
    return func(c *gin.Context) {
        caller := c.GetString("authenticated_service")
        target := c.Param("id") // container ID or name
        action := deriveAction(c.Request.Method, c.FullPath())

        allowed, trust, err := repo.CheckAccess(caller, target, action, c.Query("path"))

        repo.LogAudit(trust, caller, target, action, c.Query("path"), allowed, reason)

        if !allowed {
            c.AbortWithStatusJSON(403, gin.H{"error": "Access denied", "action": action})
            return
        }
        c.Next()
    }
}
```

#### Research References

- **Portainer**: Reverse-proxy to Docker Engine API via `/api/endpoints/{ENV_ID}/docker/...` gateway
- **Docker Socket Proxy** (Tecnativa): Boolean env var toggles per API section (CONTAINERS=1, EXEC=0, POST=1)
- **foxxmd/docker-proxy-filter**: Per-container filtering on top of socket proxy
- **Coolify**: Tiered permission levels (root, write, deploy, read, read:sensitive)
- **OPA (Open Policy Agent)**: Rego policies for fine-grained service-to-service authorization

---

### 3.2 File Injection API

#### Purpose

Read, write, list, and delete files inside running containers without rebuilding. Primary use case: updating agent instruction files (CLAUDE.md) and configuration.

#### NATS Subjects (Rust Agent)

| Subject | Direction | Description |
|---------|-----------|-------------|
| `docker.container.file.read` | Request/Reply | Read file content from container |
| `docker.container.file.write` | Request/Reply | Write file to container |
| `docker.container.file.list` | Request/Reply | List directory contents |
| `docker.container.file.delete` | Request/Reply | Delete file from container |

#### Implementation (Rust - Bollard)

Uses Docker Engine API's archive endpoints:
- **Write**: `PUT /containers/{id}/archive` -- uploads a tar archive
- **Read**: `GET /containers/{id}/archive` -- downloads a tar archive

```rust
use bollard::container::{UploadToContainerOptions, DownloadFromContainerOptions};
use tar::{Builder, Archive};

async fn write_file(
    docker: &Docker,
    container_id: &str,
    path: &str,
    content: &[u8],
) -> Result<()> {
    let (dir, filename) = split_path(path);

    let mut tar_builder = Builder::new(Vec::new());
    let mut header = tar::Header::new_gnu();
    header.set_size(content.len() as u64);
    header.set_mode(0o644);
    header.set_cksum();
    tar_builder.append_data(&mut header, filename, content)?;
    let tar_bytes = tar_builder.into_inner()?;

    docker
        .upload_to_container(
            container_id,
            Some(UploadToContainerOptions {
                path: dir,
                ..Default::default()
            }),
            tar_bytes.into(),
        )
        .await?;

    Ok(())
}

async fn read_file(
    docker: &Docker,
    container_id: &str,
    path: &str,
) -> Result<Vec<u8>> {
    let stream = docker.download_from_container(
        container_id,
        Some(DownloadFromContainerOptions {
            path: path.to_string(),
        }),
    );

    let bytes = stream.try_concat().await?;
    let mut archive = Archive::new(&bytes[..]);

    for entry in archive.entries()? {
        let mut entry = entry?;
        let mut content = Vec::new();
        entry.read_to_end(&mut content)?;
        return Ok(content);
    }

    Err(anyhow!("File not found in archive"))
}

async fn list_directory(
    docker: &Docker,
    container_id: &str,
    path: &str,
) -> Result<Vec<FileEntry>> {
    // Use docker exec to list directory
    let exec = docker
        .create_exec(
            container_id,
            CreateExecOptions {
                cmd: Some(vec!["ls", "-la", "--time-style=full-iso", path]),
                attach_stdout: Some(true),
                ..Default::default()
            },
        )
        .await?;

    let output = docker.start_exec(&exec.id, None).await?;
    parse_ls_output(output)
}
```

#### WAP HTTP Endpoints

| Method | Path | Description | Trust Check |
|--------|------|-------------|-------------|
| `GET` | `/api/v1/containers/:id/files?path=<dir>` | List directory contents | `file_read_paths` |
| `GET` | `/api/v1/containers/:id/files/read?path=<file>` | Read file content (base64 or UTF-8) | `file_read_paths` |
| `PUT` | `/api/v1/containers/:id/files/write` | Write file content | `file_write_paths` |
| `DELETE` | `/api/v1/containers/:id/files?path=<file>` | Delete file | `file_write_paths` |

#### Write File Request

```json
// PUT /api/v1/containers/armory-ideation/files/write
{
    "path": "/app/config/instructions/CLAUDE.md",
    "content": "# Agent Instructions\n\nYou are an ideation agent...",
    "encoding": "utf-8",
    "mode": "0644",
    "createDirs": true
}
```

#### List Directory Response

```json
{
    "path": "/app/config/instructions/",
    "entries": [
        {
            "name": "CLAUDE.md",
            "type": "file",
            "size": 2048,
            "mode": "0644",
            "modifiedAt": "2026-01-31T10:30:00Z"
        },
        {
            "name": "prompts/",
            "type": "directory",
            "modifiedAt": "2026-01-31T09:00:00Z"
        }
    ]
}
```

#### Research References

- **Docker Engine API**: `PUT /containers/{id}/archive` for tar-based file upload
- **Bollard SDK**: `upload_to_container` / `download_from_container` methods
- **Kubernetes ConfigMaps**: Volume-mounted configs with symlink rotation for updates
- **Stakater Reloader**: Watches ConfigMap changes and triggers pod rollouts

---

### 3.3 Agent Version Management

#### Purpose

Rolling updates for agent containers: pull new image, recreate container with same config.

#### NATS Subject

| Subject | Description |
|---------|-------------|
| `docker.container.update` | Stop -> remove -> pull -> create -> start with new image |

#### Update Flow

```
1. Client -> WAP: POST /api/v1/containers/:id/update-image
   Body: { "image": "ai-armory-client", "tag": "v2.1.0" }

2. WAP Core:
   a. Check trust: can_update_image && image matches allowed_images
   b. Log audit entry
   c. Send NATS request to agent

3. WAP Agent (Rust):
   a. docker.inspect_container(id) -> capture full config
   b. docker.pull_image(image:tag) -> pull new image
   c. docker.stop_container(id) -> graceful stop
   d. docker.remove_container(id) -> remove old
   e. docker.create_container(config + new image) -> create new
   f. docker.start_container(new_id) -> start
   g. Return new container ID

4. WAP Core -> SSE broadcast: { type: "version-update", containerId, status, newTag }
```

#### WAP HTTP Endpoints

| Method | Path | Description | Trust Check |
|--------|------|-------------|-------------|
| `POST` | `/api/v1/containers/:id/update-image` | Rolling update to new image version | `can_update_image` + `allowed_images` |
| `GET` | `/api/v1/containers/:id/env` | Get container env vars | `can_inspect` |
| `PATCH` | `/api/v1/containers/:id/env` | Update env vars (requires recreate) | `can_update_env` |

#### Update Image Request/Response

```json
// POST /api/v1/containers/armory-ideation/update-image
// Request
{
    "image": "ai-armory-client",
    "tag": "v2.1.0",
    "preserveVolumes": true,
    "preserveNetworks": true
}

// Response
{
    "success": true,
    "oldContainerId": "abc123",
    "newContainerId": "def456",
    "image": "ai-armory-client:v2.1.0",
    "status": "running"
}
```

---

### 3.4 Enhanced Health Monitoring

#### Purpose

Aggregate health, resource usage, and logs from all managed containers into a single API.

#### NATS Subjects

| Subject | Description |
|---------|-------------|
| `docker.container.health` | Single container health status |
| `docker.container.health.all` | All containers health summary |
| `docker.container.stats` | Container resource usage stats |
| `docker.container.stats.stream` | Streaming stats (for real-time dashboards) |

#### WAP HTTP Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/containers/health` | All containers health summary |
| `GET` | `/api/v1/containers/:id/health` | Single container detailed health |
| `GET` | `/api/v1/containers/:id/stats` | CPU/memory/network stats snapshot |
| `GET` | `/api/v1/containers/:id/logs?tail=100&follow=true` | Container logs (SSE stream) |

#### Health Summary Response

```json
// GET /api/v1/containers/health
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

#### Log Streaming (SSE)

```
GET /api/v1/containers/armory-ideation/logs?tail=50&follow=true

Content-Type: text/event-stream

event: log
data: {"timestamp":"2026-01-31T10:44:55Z","stream":"stdout","message":"[INFO] Message received in room cml0eixyh"}

event: log
data: {"timestamp":"2026-01-31T10:44:56Z","stream":"stdout","message":"[INFO] Invoking Claude Code CLI..."}
```

#### Research References

- **Docker Events API**: `GET /events` streaming endpoint for container lifecycle events
- **Docker Stats API**: `GET /containers/{id}/stats` for real-time resource usage
- **SSE (Server-Sent Events)**: Recommended over WebSocket for one-way server-to-client monitoring (auto-reconnect, HTTP-compatible)
- **Prometheus prom-client**: Metrics export pattern used by AI-Armory server

---

## 4. Feature 2: AI-Armory Agent Controls

### 4.1 Server API Routes

#### New Route Module: `agentControlRoutes.ts`

All routes require user authentication (SuperTokens session) and verify the user owns the container via `ContainerLaunch.accountId`.

| Method | Path | Description | WAP Proxy |
|--------|------|-------------|-----------|
| `POST` | `/v1/agents/:containerId/restart` | Restart agent container | `POST /containers/:wapId/restart` |
| `POST` | `/v1/agents/:containerId/stop` | Stop agent container | `POST /containers/:wapId/stop` |
| `POST` | `/v1/agents/:containerId/start` | Start agent container | `POST /containers/:wapId/start` |
| `POST` | `/v1/agents/:containerId/update-version` | Update agent image version | `POST /containers/:wapId/update-image` |
| `GET` | `/v1/agents/:containerId/versions` | List available image versions | Local query |
| `GET` | `/v1/agents/:containerId/files` | List instruction files | `GET /containers/:wapId/files` |
| `GET` | `/v1/agents/:containerId/files/*path` | Read file content | `GET /containers/:wapId/files/read` |
| `PUT` | `/v1/agents/:containerId/files/*path` | Write/update file | `PUT /containers/:wapId/files/write` |
| `DELETE` | `/v1/agents/:containerId/files/*path` | Delete file | `DELETE /containers/:wapId/files` |
| `GET` | `/v1/agents/health` | All agents health summary | `GET /containers/health` |
| `GET` | `/v1/agents/:containerId/health` | Single agent health | `GET /containers/:wapId/health` |
| `GET` | `/v1/agents/:containerId/stats` | Resource stats | `GET /containers/:wapId/stats` |
| `GET` | `/v1/agents/:containerId/logs` | Recent logs | `GET /containers/:wapId/logs` |
| `GET` | `/v1/agents/:containerId/env` | Get env vars (filtered) | `GET /containers/:wapId/env` |
| `PATCH` | `/v1/agents/:containerId/env` | Update env vars | `PATCH /containers/:wapId/env` |

#### WapClient Extensions

```typescript
// Add to sources/app/wap/wapClient.ts

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

#### Socket.io Events (New)

| Event | Direction | Data |
|-------|-----------|------|
| `agent-health-update` | Server -> Client | `{ containerId, health, resources }` |
| `agent-version-update` | Server -> Client | `{ containerId, oldTag, newTag, status }` |
| `agent-file-changed` | Server -> Client | `{ containerId, path, action }` |
| `agent-config-reloaded` | Server -> Client | `{ containerId, files }` |

#### Prisma Schema Extension

```prisma
model ContainerLaunch {
    // ... existing fields ...

    // New: agent access parameters (reported by container on startup)
    accessParameters   Json?     // { writablePaths, readOnlyPaths, configDir, ... }
    agentCapabilities  String[]  // ["file.read", "file.write", "restart", "hot-reload"]
    instructionsDir    String?   // "/app/config/instructions"
    supportsHotReload  Boolean   @default(false)
}
```

---

### 4.2 Frontend UI Components

#### Agent Management Dashboard

**Route:** `/projects/[projectId]/agents`

```
+---------------------------------------------+
| Agents for [Project Name]                    |
+---------------------------------------------+
| +-----------------+ +-----------------+      |
| | ideation-agent  | | backend-agent   |      |
| | * Running       | | o Stopped       |      |
| | CPU: 12% MEM: 3%| |                 |      |
| | v1.0.0          | | v1.0.0          |      |
| | Last msg: 2m ago| |                 |      |
| | [Restart] [Stop]| | [Start]         |      |
| | [Files] [Logs]  | | [Files] [Logs]  |      |
| +-----------------+ +-----------------+      |
+---------------------------------------------+
```

#### Agent File Editor

**Route:** `/projects/[projectId]/agents/[containerId]/files`

```
+------------------+---------------------------+
| File Tree        | CLAUDE.md                 |
|                  |                           |
| v config/        | # Agent Instructions      |
|   v instructions/|                           |
|     CLAUDE.md <- | You are an ideation agent |
|     prompts/     | that helps turn ideas     |
|   .env           | into product specs...     |
|                  |                           |
|                  | [Save] [Revert] [Preview] |
+------------------+---------------------------+
```

#### Agent Health Dashboard (Global)

**Route:** `/agents`

```
+---------------------------------------------+
| All Agents                   [Filter] [Sort] |
+---------------------------------------------+
| Name           | Status | CPU | MEM | Uptime |
| ideation-agent | *      | 12% | 3%  | 2h30m  |
| backend-agent  | o      | -   | -   | -      |
| docs-agent     | *      | 5%  | 2%  | 1h15m  |
+---------------------------------------------+
| Total: 3 | Running: 2 | Stopped: 1           |
+---------------------------------------------+
```

#### AgentControls Component (Reusable)

Embedded in the room page alongside the chat:

```
+------------------------------+
| Agent: ideation-agent        |
| Status: * Running  v1.0.0   |
| CPU: 12%  MEM: 245MB/8GB    |
| [Restart] [Stop] [Update v] |
| [Edit Files] [View Logs]    |
+------------------------------+
```

---

### 4.3 Agent Container Updates

#### Config Watcher Service

New file: `src/services/config-watcher.ts`

Watches the instructions directory for file changes using `chokidar` (cross-platform file watcher). When files change:

1. Detect the changed file
2. Reload system prompt from disk
3. Log the reload event
4. Report via WebSocket `machine-update` event to server

```typescript
import chokidar from 'chokidar';

class ConfigWatcher {
    private watcher: chokidar.FSWatcher;
    private instructionsDir: string;
    private onReload: (files: string[]) => void;

    constructor(
        instructionsDir: string,
        onReload: (files: string[]) => void,
    ) {
        this.instructionsDir = instructionsDir;
        this.onReload = onReload;
    }

    start() {
        this.watcher = chokidar.watch(this.instructionsDir, {
            persistent: true,
            ignoreInitial: true,
            awaitWriteFinish: { stabilityThreshold: 500 },
        });

        this.watcher.on('change', (path) => {
            logger.info(`Config file changed: ${path}`);
            this.onReload([path]);
        });

        this.watcher.on('add', (path) => {
            logger.info(`Config file added: ${path}`);
            this.onReload([path]);
        });
    }

    stop() {
        this.watcher?.close();
    }
}
```

#### Parameter Registration on Startup

When the agent container starts, it registers its access parameters:

```typescript
// In src/index.ts -- after machine registration
await serverClient.updateMachine(machineId, {
    metadata: encryptedMetadata,
    parameters: {
        writablePaths: (
            process.env.AGENT_WRITABLE_PATHS || '/app/config,/data/memory'
        ).split(','),
        readOnlyPaths: ['/app/src', '/app/dist'],
        configDir:
            process.env.AGENT_INSTRUCTIONS_DIR ||
            '/app/config/instructions',
        supportsHotReload: true,
        supportedActions: [
            'restart',
            'stop',
            'start',
            'file.read',
            'file.write',
        ],
        agentVersion: require('../package.json').version,
    },
});
```

#### New Compose Environment Variables

```yaml
environment:
    - AGENT_CONFIG_DIR=/app/config
    - AGENT_INSTRUCTIONS_DIR=/app/config/instructions
    - AGENT_WRITABLE_PATHS=/app/config,/data/memory
```

---

## 5. Implementation Roadmap

### Phase 1: Foundation (WAP Trust System)

- [ ] SQLite migration for trust_relationships + trust_audit_log tables
- [ ] Go models and repository for trust CRUD
- [ ] Trust check middleware
- [ ] Trust API endpoints (7 endpoints)
- [ ] Seed default trust for armory-server -> * (full access)

### Phase 2: File Operations (WAP + Rust Agent)

- [ ] Rust file read/write handlers using Bollard tar API
- [ ] NATS subject handlers for file operations
- [ ] Go HTTP endpoints for file CRUD (4 endpoints)
- [ ] Trust path checking with glob pattern matching

### Phase 3: Health & Monitoring (WAP)

- [ ] Container health aggregation endpoint
- [ ] Container stats endpoint (Docker Stats API)
- [ ] Container logs streaming via SSE
- [ ] NATS subjects for health queries

### Phase 4: Version Management (WAP + Rust Agent)

- [ ] Rust rolling update handler (inspect -> pull -> stop -> remove -> create -> start)
- [ ] NATS subject for container update
- [ ] Go HTTP endpoint for update-image
- [ ] Environment variable read/update endpoints

### Phase 5: AI-Armory Server Integration

- [ ] New agentControlRoutes.ts (18 endpoints)
- [ ] WapClient extensions (12 new methods)
- [ ] Prisma schema update (accessParameters, capabilities)
- [ ] Socket.io event broadcasting for agent updates

### Phase 6: AI-Armory Frontend

- [ ] Agent management dashboard page
- [ ] Agent file editor page with code editor
- [ ] Global agent health dashboard
- [ ] AgentControls reusable component
- [ ] Agent API client module

### Phase 7: Agent Container Updates

- [ ] Config watcher service (chokidar)
- [ ] Parameter registration on startup
- [ ] Hot-reload system prompt from disk
- [ ] New compose environment variables

---

## 6. Security Considerations

- All file operations are path-restricted via trust_relationships glob patterns
- Trust audit log captures every operation (allowed and denied)
- Environment variable display filters sensitive values (passwords, tokens, API keys)
- File write operations validate content size limits
- Container exec is disabled by default (most restrictive)
- Trust relationships can have expiration dates
- Rolling updates only allow pre-approved image patterns
- Agent containers run as non-root user

---

## 7. Related Projects & Applications

### AI-Armory Ecosystem

| Application | Repository | Purpose |
|-------------|-----------|---------|
| **AI-Armory Frontend** | cloud303-psupinski/ai-armory (formerly happy-tsk-next) | Next.js UI for projects, rooms, agent chat |
| **AI-Armory Server** | cloud303-psupinski/app-forge-chat (formerly happy-server) | Fastify API, container orchestration, auth |
| **WAP Docker Manager** | (private) | Go+Rust Docker management platform |
| **Agent Client** | (armory-client) | TypeScript Claude Code agent containers |
| **AI-Armory Features** | cloud303-psupinski/ai-armory-features | Feature specifications (this repo) |

### Infrastructure

| Service | Purpose |
|---------|---------|
| Traefik | Reverse proxy, TLS termination, routing |
| PostgreSQL | AI-Armory Server database |
| Redis | Cache, Socket.io adapter |
| MinIO | S3-compatible object storage |
| NATS | WAP internal message bus |
| SuperTokens | Authentication (OAuth, email/password) |
