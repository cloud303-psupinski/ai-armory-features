---
id: wap-file-injection
title: WAP File Injection API
status: draft
category: wap-proxy
priority: high
tags: [files, injection, docker, bollard, nats, tar]
depends-on: [wap-trust-system]
related: [armory-agent-api, agent-config-watcher]
owner: TBD
created: 2026-01-31
updated: 2026-01-31
---

# WAP File Injection API

## Summary

Read, write, list, and delete files inside running containers without rebuilding. Primary use case: updating agent instruction files (CLAUDE.md) and configuration. Uses Bollard's Docker tar API for file transfer and NATS for communication between WAP Core (Go) and WAP Agent (Rust). All operations are gated by the trust system's `file_read_paths` and `file_write_paths` glob patterns.

## Use Cases

- **As** a platform user, **I want** to edit agent instruction files (e.g., `CLAUDE.md`) inside a running container, **so that** I can customize agent behavior without restarting or rebuilding.
- **As** the AI-Armory Server, **I want** to list directory contents inside a container, **so that** the frontend can render a file tree for the file editor UI.
- **As** a platform operator, **I want** file writes restricted to declared paths, **so that** accidental or malicious writes to system directories are prevented.

## Functional Requirements

- **FR-001**: The system SHALL support reading file content from a running container via Docker's download-from-container API (tar archive extraction).
- **FR-002**: The system SHALL support writing file content to a running container via Docker's upload-to-container API (tar archive creation).
- **FR-003**: The system SHALL support listing directory contents inside a container via `exec` with `ls -la`.
- **FR-004**: The system SHALL support deleting files inside a container.
- **FR-005**: All file operations SHALL be routed through NATS request/reply subjects.
- **FR-006**: All file operations SHALL be gated by trust system glob-pattern matching on `file_read_paths` or `file_write_paths`.
- **FR-007**: Write operations SHALL support optional `createDirs: true` to create parent directories.
- **FR-008**: Write operations SHALL accept an `encoding` parameter (default `utf-8`) and a `mode` parameter (default `0644`).

## API Contract

### NATS Subjects (Rust Agent)

| Subject | Direction | Description |
|---------|-----------|-------------|
| `docker.container.file.read` | Request/Reply | Read file content from container |
| `docker.container.file.write` | Request/Reply | Write file to container |
| `docker.container.file.list` | Request/Reply | List directory contents |
| `docker.container.file.delete` | Request/Reply | Delete file from container |

### HTTP Endpoints

| Method | Path | Description | Trust Check |
|--------|------|-------------|-------------|
| `GET` | `/api/v1/containers/:id/files?path=<dir>` | List directory contents | `file_read_paths` |
| `GET` | `/api/v1/containers/:id/files/read?path=<file>` | Read file content (base64 or UTF-8) | `file_read_paths` |
| `PUT` | `/api/v1/containers/:id/files/write` | Write file content | `file_write_paths` |
| `DELETE` | `/api/v1/containers/:id/files?path=<file>` | Delete file | `file_write_paths` |

### Write File — Request

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

### List Directory — Response

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

## Data Model

No additional database tables. File operations are stateless pass-through to Docker API, gated by `trust_relationships.file_read_paths` and `trust_relationships.file_write_paths`.

## Implementation Notes

### Rust Bollard File Handlers

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

`[NEEDS CLARIFICATION]` Should file write operations enforce a maximum file size? If so, what is the limit?

`[NEEDS CLARIFICATION]` Should binary files be supported, or only UTF-8 text? The `encoding` field suggests text-only, but the Rust implementation handles raw bytes.

## Acceptance Criteria

- **AC-001**: Writing a file to a path matching `file_write_paths` succeeds and the file is readable afterward.
- **AC-002**: Writing a file to a path NOT matching `file_write_paths` returns 403.
- **AC-003**: Reading a file returns its content with correct encoding.
- **AC-004**: Listing a directory returns entries with name, type, size, mode, and modifiedAt.
- **AC-005**: Deleting a file removes it from the container filesystem.
- **AC-006**: Writing with `createDirs: true` creates intermediate directories.
- **AC-007**: All file operations create audit log entries via the trust system.

## Dependencies

- [WAP Trust System](../wap-trust-system/spec.md) — Required for path-based authorization.

## Research References

- **Docker Engine API**: `PUT /containers/{id}/archive` for tar-based file upload
- **Bollard SDK**: `upload_to_container` / `download_from_container` methods
- **Kubernetes ConfigMaps**: Volume-mounted configs with symlink rotation for updates
- **Stakater Reloader**: Watches ConfigMap changes and triggers pod rollouts

## Out of Scope

- Streaming large files (> 10MB) — initial implementation targets config/instruction files
- File watching/notification from inside the container (handled by [Agent Config Watcher](../agent-config-watcher/spec.md))
- Recursive directory upload
