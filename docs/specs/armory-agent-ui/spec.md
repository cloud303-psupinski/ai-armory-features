---
id: armory-agent-ui
title: AI-Armory Agent UI Components
status: draft
category: armory-frontend
priority: medium
tags: [frontend, nextjs, react, tailwind, dashboard, file-editor]
depends-on: [armory-agent-api]
related: [wap-health-monitoring, agent-config-watcher]
owner: TBD
created: 2026-01-31
updated: 2026-01-31
---

# AI-Armory Agent UI Components

## Summary

Three new pages and one reusable component in the AI-Armory Frontend (Next.js) for managing agent containers. Includes an agent management dashboard with lifecycle controls, an in-browser file editor for agent instruction files, and a health dashboard with real-time resource metrics.

## Use Cases

- **As** a platform user, **I want** a dashboard showing all my agents with their status and controls, **so that** I can manage agent lifecycle from the browser.
- **As** a platform user, **I want** an in-browser file editor for agent instruction files, **so that** I can customize agent behavior without CLI access.
- **As** a platform operator, **I want** a health dashboard showing CPU, memory, and uptime for all agents, **so that** I can monitor system resource usage.
- **As** a developer, **I want** a reusable `AgentControls` component, **so that** agent management UI can be embedded in multiple pages.

## Functional Requirements

- **FR-001**: The agent management dashboard SHALL display all agents for a project with status, version, and resource usage.
- **FR-002**: Each agent card SHALL provide lifecycle buttons: Start, Stop, Restart, and Update Version.
- **FR-003**: The file editor page SHALL display a file tree on the left and a text editor on the right.
- **FR-004**: The file editor SHALL support Save, Revert, and Preview actions.
- **FR-005**: The health dashboard SHALL display a sortable, filterable table of all agents across projects.
- **FR-006**: The `AgentControls` component SHALL be reusable and display agent status, version, resource usage, and action buttons.
- **FR-007**: All pages SHALL update in real time via Socket.io events (`agent-health-update`, `agent-version-update`, `agent-file-changed`, `agent-config-reloaded`).

## API Contract

This feature consumes the [Armory Agent API](../armory-agent-api/spec.md) endpoints. No new API surface is defined here.

### Pages and Routes

| Route | Page | Description |
|-------|------|-------------|
| `/projects/[projectId]/agents` | Agent Management Dashboard | Per-project agent list with controls |
| `/projects/[projectId]/agents/[containerId]/files` | Agent File Editor | File tree + text editor |
| `/agents` | Agent Health Dashboard (Global) | Cross-project health overview |

## Data Model

No additional data model changes. This feature consumes the Prisma schema defined in [Armory Agent API](../armory-agent-api/spec.md).

## Implementation Notes

### Agent Management Dashboard

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

### Agent File Editor

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

### Agent Health Dashboard (Global)

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

### AgentControls Component (Reusable)

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

`[NEEDS CLARIFICATION]` Which code editor library should be used for the file editor? Options include CodeMirror, Monaco Editor, or a simpler textarea with syntax highlighting.

`[NEEDS CLARIFICATION]` Should the file editor support editing multiple files simultaneously (tabs), or only one file at a time?

## Acceptance Criteria

- **AC-001**: The agent management dashboard loads and displays all agents for the current project.
- **AC-002**: Clicking Restart/Stop/Start triggers the corresponding API call and updates the UI on success.
- **AC-003**: The file editor displays the file tree from the container's config directory.
- **AC-004**: Saving a file in the editor writes it to the container and shows a success notification.
- **AC-005**: The health dashboard displays all agents across all of the user's projects.
- **AC-006**: Real-time Socket.io events update the UI without manual refresh.
- **AC-007**: The `AgentControls` component renders correctly when embedded in different page contexts.

## Dependencies

- [Armory Agent API](../armory-agent-api/spec.md) — All data fetching and mutations.

## Research References

- **Next.js App Router**: File-based routing with dynamic segments
- **CodeMirror 6**: Extensible code editor for the web
- **Tailwind CSS**: Utility-first styling consistent with existing frontend

## Out of Scope

- Terminal/shell access to containers from the UI
- Drag-and-drop file upload
- Agent creation wizard (containers are created through the existing project setup flow)
- Mobile-responsive layout (desktop-first)
