---
id: wap-version-management
title: WAP Agent Version Management
status: draft
category: wap-proxy
priority: medium
tags: [versioning, rolling-update, docker, nats, environment]
depends-on: [wap-trust-system]
related: [wap-health-monitoring, armory-agent-api]
owner: TBD
created: 2026-01-31
updated: 2026-01-31
---

# WAP Agent Version Management

## Summary

Rolling updates for agent containers: pull new image, recreate container with same config. Also exposes environment variable read/write endpoints for runtime configuration. All operations are gated by the trust system's `can_update_image`, `allowed_images`, and `can_update_env` permissions.

## Use Cases

- **As** a platform operator, **I want** to update an agent container to a new image tag, **so that** I can deploy bug fixes and new capabilities without manual Docker commands.
- **As** the AI-Armory Server, **I want** to read and update environment variables on a running container, **so that** the frontend can expose a configuration UI.
- **As** a security reviewer, **I want** image updates restricted to a pre-approved list, **so that** arbitrary images cannot be deployed.

## Functional Requirements

- **FR-001**: The system SHALL perform a rolling update: inspect existing config, pull new image, stop old container, remove old container, create new container with same config + new image, start new container.
- **FR-002**: The rolling update SHALL preserve volumes and network configuration by default.
- **FR-003**: The system SHALL broadcast an SSE event on version update status changes.
- **FR-004**: The system SHALL validate the requested image:tag against the trust relationship's `allowed_images` list.
- **FR-005**: The system SHALL provide endpoints to read and update container environment variables.
- **FR-006**: Environment variable updates SHALL require the `can_update_env` trust permission.

## API Contract

### NATS Subject

| Subject | Description |
|---------|-------------|
| `docker.container.update` | Stop -> remove -> pull -> create -> start with new image |

### HTTP Endpoints

| Method | Path | Description | Trust Check |
|--------|------|-------------|-------------|
| `POST` | `/api/v1/containers/:id/update-image` | Rolling update to new image version | `can_update_image` + `allowed_images` |
| `GET` | `/api/v1/containers/:id/env` | Get container env vars | `can_inspect` |
| `PATCH` | `/api/v1/containers/:id/env` | Update env vars (requires recreate) | `can_update_env` |

### Update Image — Request/Response

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

### Update Flow

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

## Data Model

No additional database tables. The trust relationship's `can_update_image`, `allowed_images`, and `can_update_env` fields control authorization.

## Implementation Notes

- The Rust agent must capture the full container config (env, volumes, networks, labels, restart policy) before removal to recreate it with the new image.
- `preserveVolumes` and `preserveNetworks` default to `true` to prevent data loss.
- The SSE broadcast should include intermediate statuses: `pulling`, `stopping`, `removing`, `creating`, `starting`, `completed`, `failed`.

`[NEEDS CLARIFICATION]` What happens if the image pull fails? Should the system roll back to the original container, or leave it in a stopped state?

`[NEEDS CLARIFICATION]` Should there be a timeout on the rolling update operation? If so, what is the default?

## Acceptance Criteria

- **AC-001**: Updating to an image in `allowed_images` succeeds and the container runs the new version.
- **AC-002**: Updating to an image NOT in `allowed_images` returns 403.
- **AC-003**: Volumes mounted on the old container are present on the new container.
- **AC-004**: Network connections are preserved after update.
- **AC-005**: An SSE event is broadcast with the update result.
- **AC-006**: Reading env vars returns the container's current environment.
- **AC-007**: Updating env vars without `can_update_env` returns 403.

## Dependencies

- [WAP Trust System](../wap-trust-system/spec.md) — Required for `can_update_image`, `allowed_images`, `can_update_env` checks.

## Research References

- **Watchtower**: Automated Docker container updates with similar inspect-pull-recreate flow
- **Docker Compose**: `docker compose up -d` performs equivalent rolling updates

## Out of Scope

- Multi-container coordinated updates (e.g., updating server + agent together)
- Automatic rollback on health check failure post-update
- Image registry authentication management
