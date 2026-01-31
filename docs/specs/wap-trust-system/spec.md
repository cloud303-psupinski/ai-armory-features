---
id: wap-trust-system
title: WAP Trust Relationship System
status: draft
category: wap-proxy
priority: high
tags: [trust, authorization, sqlite, audit, middleware]
depends-on: []
related: [wap-file-injection, wap-version-management, wap-health-monitoring]
owner: TBD
created: 2026-01-31
updated: 2026-01-31
---

# WAP Trust Relationship System

## Summary

A fine-grained, per-container permission model that controls which services can perform which actions on which containers. All access decisions are recorded in an audit log. This is the foundational security layer that all other WAP features depend on.

## Use Cases

- **As** the AI-Armory Server, **I want** to declare which containers I can manage and what operations I'm allowed to perform, **so that** container access is explicitly authorized rather than implicitly open.
- **As** a platform operator, **I want** an audit trail of every trust-checked action, **so that** I can investigate incidents and verify compliance.
- **As** a security reviewer, **I want** trust relationships to support expiration dates, **so that** temporary access grants don't persist indefinitely.

## Functional Requirements

- **FR-001**: The system SHALL store trust relationships in a SQLite table with a unique constraint on `(caller_service, target_container)`.
- **FR-002**: Each trust relationship SHALL define boolean permissions for: inspect, start, stop, restart, remove, exec, logs, stats, update_env, update_image.
- **FR-003**: Each trust relationship SHALL define glob-pattern arrays for `file_read_paths` and `file_write_paths`.
- **FR-004**: Each trust relationship SHALL define an `allowed_images` array for version update authorization.
- **FR-005**: The system SHALL provide a trust check endpoint that evaluates caller + target + action + resourcePath and returns an allow/deny decision.
- **FR-006**: Every trust check SHALL be recorded in a `trust_audit_log` table.
- **FR-007**: Trust relationships SHALL support optional `expires_at` timestamps.
- **FR-008**: A Gin middleware SHALL intercept container-targeted requests and enforce trust checks before handler execution.

## API Contract

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/trust` | List all trust relationships |
| POST | `/api/v1/trust` | Create trust relationship |
| GET | `/api/v1/trust/:id` | Get specific relationship |
| PATCH | `/api/v1/trust/:id` | Update permissions |
| DELETE | `/api/v1/trust/:id` | Revoke trust |
| GET | `/api/v1/trust/audit` | Query audit log |
| POST | `/api/v1/trust/check` | Check if action is allowed |

### Trust Check — Request/Response

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

## Data Model

### SQLite Schema

```sql
CREATE TABLE trust_relationships (
    id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    caller_service TEXT NOT NULL,
    target_container TEXT NOT NULL,
    can_inspect BOOLEAN DEFAULT 1,
    can_start BOOLEAN DEFAULT 0,
    can_stop BOOLEAN DEFAULT 0,
    can_restart BOOLEAN DEFAULT 0,
    can_remove BOOLEAN DEFAULT 0,
    can_exec BOOLEAN DEFAULT 0,
    can_logs BOOLEAN DEFAULT 1,
    can_stats BOOLEAN DEFAULT 1,
    file_read_paths TEXT DEFAULT '[]',
    file_write_paths TEXT DEFAULT '[]',
    can_update_env BOOLEAN DEFAULT 0,
    can_update_image BOOLEAN DEFAULT 0,
    allowed_images TEXT DEFAULT '[]',
    description TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    expires_at DATETIME,
    UNIQUE(caller_service, target_container)
);

CREATE TABLE trust_audit_log (
    id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
    trust_id TEXT REFERENCES trust_relationships(id) ON DELETE SET NULL,
    caller_service TEXT NOT NULL,
    target_container TEXT NOT NULL,
    action TEXT NOT NULL,
    resource_path TEXT,
    allowed BOOLEAN NOT NULL,
    reason TEXT,
    request_metadata TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Implementation Notes

### Trust Check Middleware (Go)

```go
func TrustCheckMiddleware(repo *repository.TrustRepository) gin.HandlerFunc {
    return func(c *gin.Context) {
        caller := c.GetString("authenticated_service")
        target := c.Param("id")
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

`[NEEDS CLARIFICATION]` How is `deriveAction` implemented? The mapping from HTTP method + path to action string (e.g., `file.write`, `container.restart`) needs to be defined.

## Acceptance Criteria

- **AC-001**: Running the SQLite migration creates both `trust_relationships` and `trust_audit_log` tables without error.
- **AC-002**: Creating a trust relationship with duplicate `(caller_service, target_container)` returns a 409 Conflict.
- **AC-003**: A `POST /api/v1/trust/check` request with a valid caller/target/action returns `{ "allowed": true }` when a matching trust exists.
- **AC-004**: A denied trust check returns `{ "allowed": false }` and creates an audit log entry with `allowed = false`.
- **AC-005**: An expired trust relationship (past `expires_at`) is treated as denied.
- **AC-006**: The middleware blocks requests to container endpoints when no matching trust relationship exists.
- **AC-007**: The audit log endpoint supports filtering by `caller_service`, `target_container`, `action`, and date range.

## Dependencies

None — this is the foundational feature.

## Research References

- **Portainer**: Reverse-proxy to Docker Engine API with RBAC
- **Docker Socket Proxy** (Tecnativa): Boolean env var toggles per API section
- **foxxmd/docker-proxy-filter**: Per-container request filtering
- **Coolify**: Tiered permission levels for container management
- **OPA (Open Policy Agent)**: Rego-based policy evaluation for fine-grained authorization

## Out of Scope

- Multi-tenant trust isolation (single-tenant deployment assumed)
- OAuth2/OIDC integration for service-to-service auth (HTTP Basic Auth is used)
- Trust relationship UI in the frontend (API-only for now)
