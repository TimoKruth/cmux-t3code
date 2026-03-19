# cmux + t3code Unification Design

## Overview

Combine cmux (native macOS terminal app) and t3code (web-based coding agent GUI) into a single application. cmux is the host; t3code runs as a sidecar service providing chat UI and agent orchestration.

The unified app gives each project a workspace with collapsible task-based "writers," where each writer provides a t3code chat session as its main view and optional Ghostty terminal splits.

## Architecture

```
cmux (Swift/AppKit) — Host Application
│
├── Sidebar (refactored from VerticalTabsSidebar in ContentView.swift)
│   ├── Workspace (Project) — collapsible
│   │   ├── Writer (Task/Topic)
│   │   ├── Writer
│   │   └── + New writer
│   └── Workspace — collapsed
│
├── Main View (per writer)
│   ├── CmuxWebView (existing WKWebView subclass) — t3code React UI, persistent root
│   └── Ghostty Terminal splits via Bonsplit (Cmd+D / Cmd+Shift+D)
│
└── Sidecar Manager
    └── Spawns one t3code Node.js server per workspace
        └── ws://localhost:{PORT}

t3code (Node.js) — Sidecar Service (one per workspace)
├── WebSocket server (existing JSON-RPC API, single connection per client)
├── Orchestration engine (existing)
├── Provider: Codex (existing)
└── SQLite persistence at {project-dir}/.cmux/state.sqlite
```

### Existing Infrastructure to Reuse

- **CmuxWebView** (`Sources/Panels/CmuxWebView.swift`): Custom WKWebView subclass already handling Cmd-key routing, focus management, and middle-click. Used by BrowserPanel today — reuse for chat panels.
- **Bonsplit** (`Sources/Workspace.swift`): Recursive split tree library already managing pane splits. Extend to support chat WebView as a pane type alongside terminal panes.
- **SessionPersistence** (`Sources/SessionPersistence.swift`): JSON snapshot system saving to `~/Library/Application Support/cmux/session-{bundleId}.json`. Extend snapshot schemas to include writer data.
- **VerticalTabsSidebar** (`Sources/ContentView.swift`): Current sidebar — refactor from flat tab list to two-level workspace → writer tree.

### Communication Flow

1. cmux spawns t3code server for each workspace, passing `--state-dir {project-dir}/.cmux` (t3code creates `state.sqlite` inside)
2. t3code server binds to a random localhost port; cmux detects readiness by parsing structured Effect log output from stdout (contains port in "T3 Code running" message)
3. Each writer's CmuxWebView loads `http://localhost:{PORT}/#{threadId}` (TanStack Router hash-based routing)
4. WebView opens a single WebSocket to the t3code server; thread/session selection happens at the application level via JSON-RPC messages over that connection (not via URL params)
5. Terminal splits are native Ghostty panes via Bonsplit, managed by cmux — no t3code involvement

## Sidebar Model

### Level 1 — Workspace (Project)

- Represents a project directory / git repo
- Collapsible row: chevron, color indicator, project name, git branch
- Collapsed state shows writer count badge
- Click toggles writer list visibility

### Level 2 — Writer (Task/Topic)

- Nested under workspace, indented
- Shows: writer name (user-editable), active status indicator
- Click switches main view to that writer
- Supports: rename, reorder (drag), delete (with confirmation)
- "+ New writer" button at bottom of each workspace's list

### Data Model (Swift)

```swift
struct Workspace {
    let id: UUID
    var name: String
    var directory: URL
    var gitBranch: String?
    var isExpanded: Bool
    var writers: [Writer]
}

struct Writer {
    let id: UUID
    var name: String
    var t3codeThreadId: String    // maps to t3code thread (used in URL hash and WS messages)
    var terminalSplits: [SplitConfig]
    var isActive: Bool
}
```

cmux persists this in its existing JSON snapshot system (`~/Library/Application Support/cmux/session-{bundleId}.json`). The `t3codeThreadId` bridges to t3code's SQLite session.

## Writer Main View & Splits

### Default State

Clicking a writer shows a single CmuxWebView loading:
```
http://localhost:{PORT}/#{t3codeThreadId}
```
The WebView opens one WebSocket to the sidecar. Thread selection is handled at the app level — the WebView's React UI navigates to the correct thread via TanStack Router.

### Terminal Splits

- Cmd+D: vertical split (terminal below chat)
- Cmd+Shift+D: horizontal split (terminal to the right)
- Uses cmux's existing Bonsplit-based split tree infrastructure
- Terminal working directory defaults to workspace's project directory
- Multiple terminal splits allowed (same as cmux today)

### Closing Behavior

- Cmd+W on focused terminal: removes that pane only
- Chat WebView cannot be closed — it is the writer's root
- Closing last terminal returns to chat-only view
- Deleting a writer from sidebar tears down WebView + all terminal splits

### Layout Persistence

cmux stores per-writer: which splits exist, orientation, size ratios. On restore, cmux recreates WebView + terminal splits from saved layout.

## Sidecar Lifecycle

### Startup

1. cmux launches and reads persisted workspace snapshots from `~/Library/Application Support/cmux/`
2. For each workspace, spawns: `node apps/server/dist/index.js --state-dir {project-dir}/.cmux --auto-bootstrap-project-from-cwd` with `cwd` set to the workspace's project directory
3. t3code server binds to an available port; cmux parses the structured "T3 Code running" log line from stdout to extract the port number
4. cmux stores the port, uses it for all CmuxWebView URLs in that workspace

### One Server Per Workspace

Each workspace runs its own isolated t3code server process. This provides:
- Clean isolation between projects
- Independent SQLite databases
- Independent provider (Codex) processes
- Simple crash recovery (only affects one workspace)

### Session Creation

1. User clicks "+ New writer"
2. cmux creates a Writer entry and opens a CmuxWebView to the t3code UI
3. User starts a conversation → t3code creates a new thread, returns the threadId
4. cmux stores the threadId in the Writer model for future session restoration

### Health & Recovery

- cmux monitors each sidecar process
- If sidecar crashes: cmux respawns it automatically
- WebViews detect WebSocket disconnect, show "Reconnecting..." state
- On reconnect: sessions resume from SQLite — no data loss

### Full Restart Recovery

On cmux restart:
- cmux reads persisted workspace config (project directories, writer-to-session-ID mappings)
- Spawns t3code servers pointing at existing SQLite files
- WebViews load with same session IDs
- All chat history, orchestration events, and checkpoints are intact

**Recovers:** chat messages, conversation history, orchestration state, git checkpoints, writer names/ordering/sidebar state, split layouts

**Does not recover:** active terminal shell sessions (new shells open in correct working directory), in-flight provider operations (orchestration state shows where it stopped)

### Shutdown

- cmux quits → SIGTERM to all sidecar processes
- t3code flushes SQLite → clean exit
- If sidecar doesn't exit within 5s → SIGKILL

## Persistence Strategy

Split ownership — each system persists what it's good at:

| Data | Owner | Storage |
|------|-------|---------|
| Workspace list, names, directories | cmux | Existing workspace persistence |
| Writer list, names, ordering | cmux | Existing workspace persistence |
| Sidebar collapse/expand state | cmux | Existing workspace persistence |
| Split layout (orientation, sizes) | cmux | Existing workspace persistence |
| Chat messages, conversation history | t3code | `{project-dir}/.cmux/state.sqlite` |
| Orchestration events, turn state | t3code | `{project-dir}/.cmux/state.sqlite` |
| Git checkpoints | t3code | Git (on disk) |
| Provider state | t3code | `{project-dir}/.cmux/state.sqlite` |

## Required Code Changes

### cmux (Swift) — Modifications

1. **Sidebar refactor** (`ContentView.swift: VerticalTabsSidebar`) — Replace flat `LazyVStack` of `TabItemView` with two-level workspace → writer tree view. Add collapse/expand, writer CRUD, drag-to-reorder.

2. **`Writer` model** — New data structure: thread ID, name, split layout config. Add to `SessionWorkspaceSnapshot` in `SessionPersistence.swift`.

3. **`WriterContentView`** — New container view: `CmuxWebView` (chat) as root pane in a Bonsplit split tree + Ghostty terminal panes as splits. Extend `Workspace.swift`'s panel system to support a chat panel type alongside terminal panels.

4. **`T3CodeSidecarManager`** — New class. Per workspace: spawn Node.js process via `Process()`, parse Effect structured log from stdout to extract port, monitor process health via `terminationHandler`, handle restart/shutdown.

5. **WebView integration** — Reuse `CmuxWebView` from `Panels/CmuxWebView.swift`. Load t3code React UI. Minimal native-to-JS bridge via `WKScriptMessageHandler`: thread ID for session restoration, theme sync.

6. **Persistence update** — Extend `SessionWorkspaceSnapshot` and `SessionPanelSnapshot` in `SessionPersistence.swift` to include writer list, thread IDs, and chat panel snapshots. Autosave interval already 8s.

### t3code (TypeScript) — Modifications

1. **`--state-dir` already exists** — cmux passes `--state-dir {project-dir}/.cmux` so t3code writes `state.sqlite` there. No new flag needed.

2. **Session URL routing** — `/$threadId` route already exists in `apps/web/src/routes/_chat.$threadId.tsx` via TanStack Router. Verify it works for direct-loading via hash URL from a WebView.

3. **Port reporting** — The server already logs port info in the "T3 Code running" Effect log. May need a machine-readable startup signal (e.g., a JSON line on stdout) for reliable parsing by cmux's sidecar manager. Alternatively, cmux can parse the structured log format.

### No Changes Required

- Ghostty terminal rendering (libghostty)
- Bonsplit split tree library
- Provider abstraction (Codex works as-is)
- t3code orchestration engine
- t3code contracts/shared packages
- cmux keyboard shortcuts (Cmd+D split reused)
- CmuxWebView subclass (reused as-is)

## V1 Scope

### In Scope

1. Sidebar with workspace → writer collapsible hierarchy
2. Writer main view with t3code chat (WKWebView)
3. Terminal splits via Cmd+D / Cmd+Shift+D (Ghostty)
4. t3code sidecar lifecycle (spawn, health, recovery, shutdown)
5. Split persistence (cmux layout + t3code sessions)
6. Full restart recovery
7. Codex as provider
8. Writer CRUD (create, rename, reorder, delete)

### Out of Scope (Future)

- Additional providers (Claude Code, Aider, etc.)
- Remote SSH integration with writers
- Cross-writer context sharing
- cmux browser panel integration with chat
- Notification bridging (t3code events → cmux notifications)
- Native Swift chat UI (replacing WebView)
