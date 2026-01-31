# AI-Armory Features

Feature specifications for the AI-Armory platform — enabling frontend management of AI agent containers through WAP (Web Application Platform) Docker Manager.

The platform provides trust-based access control, file injection into running containers, real-time health monitoring, version management, and environment configuration for AI agents powered by Claude Code.

## Architecture

| Component | Stack | Role |
|-----------|-------|------|
| WAP Docker Manager | Go + Rust + NATS + SQLite | Container orchestration |
| AI-Armory Server | Fastify + Prisma + PostgreSQL | Application backend |
| AI-Armory Frontend | Next.js + Tailwind CSS | User interface |
| Agent Containers | TypeScript + Claude Code CLI | AI agent runtime |

See [docs/architecture.md](docs/architecture.md) for integration flow and infrastructure details.

## Feature Registry

| Status | Feature | Description | Category | Dependencies |
|--------|---------|-------------|----------|--------------|
| `[DRAFT]` | [WAP Trust System](docs/specs/wap-trust-system/spec.md) | Per-container permission model with audit logging | wap-proxy | — |
| `[DRAFT]` | [WAP File Injection](docs/specs/wap-file-injection/spec.md) | Read/write/list/delete files inside running containers | wap-proxy | Trust System |
| `[DRAFT]` | [WAP Version Management](docs/specs/wap-version-management/spec.md) | Rolling container updates and env var management | wap-proxy | Trust System |
| `[DRAFT]` | [WAP Health Monitoring](docs/specs/wap-health-monitoring/spec.md) | Aggregated health, resource stats, and log streaming | wap-proxy | Trust System |
| `[DRAFT]` | [Armory Agent API](docs/specs/armory-agent-api/spec.md) | Server-side proxy routes for all agent operations (15 endpoints) | armory-server | All WAP features |
| `[DRAFT]` | [Armory Agent UI](docs/specs/armory-agent-ui/spec.md) | Dashboard, file editor, and health views | armory-frontend | Agent API |
| `[DRAFT]` | [Agent Config Watcher](docs/specs/agent-config-watcher/spec.md) | Hot-reload agent instructions on file changes | agent-container | File Injection |

## Implementation Roadmap

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

## Related Projects

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

## Security Considerations

- All file operations are path-restricted via trust_relationships glob patterns
- Trust audit log captures every operation (allowed and denied)
- Environment variable display filters sensitive values (passwords, tokens, API keys)
- File write operations validate content size limits
- Container exec is disabled by default (most restrictive)
- Trust relationships can have expiration dates
- Rolling updates only allow pre-approved image patterns
- Agent containers run as non-root user

## Documentation

- [Architecture Overview](docs/architecture.md)
- [Documentation Agent Instructions](Agent_Documentation_1.0.md)
- [Legacy Feature Spec](wrp_features.md) — Original monolithic specification (kept for reference)
