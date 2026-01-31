---
id: agent-config-watcher
title: Agent Container Config Watcher
status: draft
category: agent-container
priority: medium
tags: [chokidar, hot-reload, config, watcher, typescript]
depends-on: [wap-file-injection]
related: [armory-agent-api, armory-agent-ui]
owner: TBD
created: 2026-01-31
updated: 2026-01-31
---

# Agent Container Config Watcher

## Summary

A file-watching service running inside each agent container that detects changes to instruction files (written via the [File Injection API](../wap-file-injection/spec.md)) and triggers a hot-reload of the agent's system prompt. Reports reload events via WebSocket to the Armory Server. Also includes parameter registration on startup so the server knows what each agent supports.

## Use Cases

- **As** a platform user, **I want** my agent to pick up instruction file changes immediately, **so that** editing `CLAUDE.md` via the file editor takes effect without restarting the container.
- **As** the AI-Armory Server, **I want** each agent to register its capabilities and writable paths on startup, **so that** the frontend can show the correct controls for each agent.
- **As** a developer, **I want** config file change events logged and reported, **so that** I can verify hot-reload is working correctly.

## Functional Requirements

- **FR-001**: The service SHALL watch the instructions directory using `chokidar` with `awaitWriteFinish` to debounce rapid writes.
- **FR-002**: On file change or addition, the service SHALL invoke a reload callback with the list of changed file paths.
- **FR-003**: The service SHALL report reload events via WebSocket `machine-update` event to the Armory Server.
- **FR-004**: On startup, the agent SHALL register parameters (writable paths, read-only paths, config directory, supported actions, agent version) with the Armory Server.
- **FR-005**: The `AGENT_INSTRUCTIONS_DIR`, `AGENT_CONFIG_DIR`, and `AGENT_WRITABLE_PATHS` environment variables SHALL configure the watcher and registration.

## API Contract

No new HTTP endpoints. This feature runs inside the agent container and communicates via the existing WebSocket `machine-update` channel.

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AGENT_CONFIG_DIR` | `/app/config` | Root config directory |
| `AGENT_INSTRUCTIONS_DIR` | `/app/config/instructions` | Directory to watch for instruction file changes |
| `AGENT_WRITABLE_PATHS` | `/app/config,/data/memory` | Comma-separated list of writable paths |

### Docker Compose Configuration

```yaml
environment:
    - AGENT_CONFIG_DIR=/app/config
    - AGENT_INSTRUCTIONS_DIR=/app/config/instructions
    - AGENT_WRITABLE_PATHS=/app/config,/data/memory
```

## Data Model

No database changes. Parameters are registered via the existing `serverClient.updateMachine()` API.

### Parameter Registration Payload

```typescript
{
    writablePaths: ["/app/config", "/data/memory"],
    readOnlyPaths: ["/app/src", "/app/dist"],
    configDir: "/app/config/instructions",
    supportsHotReload: true,
    supportedActions: ["restart", "stop", "start", "file.read", "file.write"],
    agentVersion: "1.0.0"
}
```

## Implementation Notes

### ConfigWatcher Service

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

### Parameter Registration on Startup

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

`[NEEDS CLARIFICATION]` How does the reload callback actually update the system prompt in the running Claude Code CLI session? Is it a process signal, IPC message, or file re-read on next invocation?

`[NEEDS CLARIFICATION]` Should the watcher also handle file deletions (e.g., removing an instruction file)?

## Acceptance Criteria

- **AC-001**: Modifying a file in the instructions directory triggers the reload callback within 1 second.
- **AC-002**: Adding a new file to the instructions directory triggers the reload callback.
- **AC-003**: The `machine-update` WebSocket event is sent to the Armory Server on reload.
- **AC-004**: On startup, the agent registers its parameters with the Armory Server.
- **AC-005**: The `AGENT_INSTRUCTIONS_DIR` environment variable correctly configures the watch path.
- **AC-006**: Rapid successive writes are debounced (only one reload per stabilization window).

## Dependencies

- [WAP File Injection](../wap-file-injection/spec.md) — Files are written into the container via this API, triggering the watcher.

## Research References

- **chokidar**: Cross-platform file watching library for Node.js
- **nodemon**: Similar watch-and-restart pattern for development servers

## Out of Scope

- Watching for changes outside the instructions directory
- Automatic agent restart on config change (hot-reload only)
- Config file validation or schema enforcement
- Syncing config changes back to a git repository
