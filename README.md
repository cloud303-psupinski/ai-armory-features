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

| Phase | Scope | Key Deliverables |
|-------|-------|------------------|
| 1 | WAP Trust System | SQLite migration, Go middleware, 7 API endpoints |
| 2 | WAP File Operations | Rust Bollard handlers, NATS subjects, 4 HTTP endpoints |
| 3 | WAP Health & Monitoring | Health aggregation, stats, SSE log streaming |
| 4 | WAP Version Management | Rust rolling-update handler, env var endpoints |
| 5 | AI-Armory Server Integration | 15 route proxies, WapClient extensions, Prisma migration |
| 6 | AI-Armory Frontend | Agent dashboard, file editor, health dashboard |
| 7 | Agent Container Updates | Config watcher, parameter registration, hot-reload |

## Related Projects

| Application | Repository | Purpose |
|-------------|-----------|---------|
| AI-Armory Frontend | cloud303-psupinski/ai-armory | Next.js UI |
| AI-Armory Server | cloud303-psupinski/app-forge-chat | Fastify API |
| WAP Docker Manager | (private) | Go+Rust Docker management |
| Agent Client | (armory-client) | TypeScript Claude Code agents |
| AI-Armory Features | cloud303-psupinski/ai-armory-features | This repo — feature specs |

## Security Principles

- File operations restricted to declared glob patterns via trust system
- Audit log records every access-checked operation
- Environment variable display filters sensitive values
- Container exec disabled by default
- Trust relationships support expiration dates
- Rolling updates restricted to pre-approved images
- Agent containers run as non-root

## Documentation

- [Architecture Overview](docs/architecture.md)
- [Documentation Agent Instructions](Agent_Documentation_1.0.md)
- [Legacy Feature Spec](wrp_features.md) — Original monolithic specification (kept for reference)
