# cmux + t3code Unification Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate t3code's AI chat/orchestration into cmux as a sidecar service, with a two-level sidebar (workspace → writer) where each writer provides a persistent chat panel + optional Ghostty terminal splits.

**Architecture:** cmux (Swift/AppKit) hosts everything. Each workspace spawns its own t3code Node.js server process. Writers are nested under workspaces in the sidebar; each writer's main view is a CmuxWebView showing t3code's React UI, with Bonsplit-managed Ghostty terminal splits. Persistence is split: cmux owns layout/sidebar, t3code owns chat/sessions via SQLite.

**Tech Stack:** Swift/AppKit (cmux), TypeScript/Node.js/Effect (t3code), WKWebView, Bonsplit, libghostty, SQLite, WebSocket

**Spec:** `docs/superpowers/specs/2026-03-18-cmux-t3code-unification-design.md`

---

## File Structure

### New Files (cmux - Swift)

| File | Responsibility |
|------|---------------|
| `Sources/Writer.swift` | Writer data model (id, name, threadId, splitConfig) |
| `Sources/WriterSidebarView.swift` | Two-level sidebar: workspace → writer tree |
| `Sources/WriterContentView.swift` | Container for chat WebView + terminal splits |
| `Sources/Panels/ChatPanel.swift` | Chat panel wrapping CmuxWebView for t3code UI |
| `Sources/T3CodeSidecarManager.swift` | Spawns/monitors/restarts t3code Node.js per workspace |

### Modified Files (cmux - Swift)

| File | Changes |
|------|---------|
| `Sources/SessionPersistence.swift` | Add `SessionWriterSnapshot`, extend `SessionWorkspaceSnapshot` |
| `Sources/ContentView.swift` | Replace `VerticalTabsSidebar` body to use `WriterSidebarView` |
| `Sources/Workspace.swift` | Add writer management, extend `createPanel()` for `.chat` type |
| `Sources/AppDelegate.swift` | Integrate sidecar lifecycle into app startup/shutdown |

### Modified Files (t3code - TypeScript)

| File | Changes |
|------|---------|
| `apps/server/src/main.ts` | Add machine-readable JSON startup line on stdout |

### No Changes Needed

| File | Why |
|------|-----|
| `Sources/Panels/CmuxWebView.swift` | Reused as-is for chat panels |
| `apps/server/src/config.ts` | `--state-dir` flag already exists |
| `apps/server/src/persistence/Layers/Sqlite.ts` | Already uses `stateDir` to construct DB path |
| `apps/web/src/routes/_chat.$threadId.tsx` | Thread routing already works |
| `apps/web/src/wsTransport.ts` | WebSocket URL auto-derives from page origin |

---

## Task 1: t3code Machine-Readable Startup Signal

**Why first:** cmux's sidecar manager needs to reliably detect when t3code is ready and on which port. The existing Effect structured log is hard to parse from Swift. Add a simple JSON line.

**Files:**
- Modify: `t3code/apps/server/src/main.ts:265-269`
- Test: Manual — start server, verify JSON line appears on stdout

- [ ] **Step 1: Add JSON startup line to main.ts**

In `t3code/apps/server/src/main.ts`, find the "T3 Code running" log block (lines 265-269). Add a machine-readable JSON line to stdout immediately before the Effect log:

```typescript
// Add right before the Effect.logInfo call at line 265
const startupSignal = JSON.stringify({
  type: "server-started",
  port: safeConfig.port,
  host: safeConfig.host ?? "127.0.0.1",
  stateDir: safeConfig.stateDir,
  cwd: safeConfig.cwd,
});
process.stdout.write(startupSignal + "\n");
```

Context: `safeConfig` is defined earlier in the same function as the config object with sensitive fields removed. The `port` field is the resolved port number. This line must come AFTER the HTTP server is listening but BEFORE the `Effect.logInfo` call.

- [ ] **Step 2: Verify the startup signal**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/t3code
bun run build --filter=t3
node apps/server/dist/index.mjs --state-dir /tmp/t3test --port 0 2>/dev/null | head -1
```

Expected: A single JSON line like:
```json
{"type":"server-started","port":3773,"host":"127.0.0.1","stateDir":"/tmp/t3test","cwd":"/Users/..."}
```

- [ ] **Step 3: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/t3code
git add apps/server/src/main.ts
git commit -m "feat: add machine-readable JSON startup signal on stdout

cmux sidecar manager will parse this to detect server readiness and port."
```

---

## Task 2: Writer Data Model

**Why:** All subsequent tasks depend on the Writer struct and its persistence schema.

**Files:**
- Create: `cmux/Sources/Writer.swift`
- Modify: `cmux/Sources/SessionPersistence.swift:330-343` (SessionWorkspaceSnapshot)

- [ ] **Step 1: Create Writer.swift**

Create `cmux/Sources/Writer.swift`:

```swift
import Foundation
import Combine

/// A writer represents a task/topic within a workspace.
/// Each writer has a t3code chat session and optional terminal splits.
final class Writer: ObservableObject, Identifiable, Codable {
    let id: UUID
    @Published var name: String
    @Published var t3codeThreadId: String?
    @Published var isActive: Bool

    init(id: UUID = UUID(), name: String, t3codeThreadId: String? = nil) {
        self.id = id
        self.name = name
        self.t3codeThreadId = t3codeThreadId
        self.isActive = false
    }

    // MARK: - Codable

    enum CodingKeys: String, CodingKey {
        case id, name, t3codeThreadId, isActive
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.id = try container.decode(UUID.self, forKey: .id)
        self.name = try container.decode(String.self, forKey: .name)
        self.t3codeThreadId = try container.decodeIfPresent(String.self, forKey: .t3codeThreadId)
        self.isActive = try container.decodeIfPresent(Bool.self, forKey: .isActive) ?? false
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(name, forKey: .name)
        try container.encodeIfPresent(t3codeThreadId, forKey: .t3codeThreadId)
        try container.encode(isActive, forKey: .isActive)
    }
}
```

- [ ] **Step 2: Add SessionWriterSnapshot to SessionPersistence.swift**

In `cmux/Sources/SessionPersistence.swift`, add the writer snapshot struct right before `SessionPanelSnapshot` (before line 243):

```swift
/// Snapshot of a single writer (task/topic) within a workspace.
struct SessionWriterSnapshot: Codable, Equatable {
    let id: UUID
    var name: String
    var t3codeThreadId: String?
    var layout: SessionWorkspaceLayoutSnapshot?
}
```

- [ ] **Step 3: Extend SessionWorkspaceSnapshot with writers**

In `cmux/Sources/SessionPersistence.swift`, find `SessionWorkspaceSnapshot` (lines 330-343). Add a `writers` field and an `activeWriterId` field:

```swift
// Add to SessionWorkspaceSnapshot's properties:
var writers: [SessionWriterSnapshot]?
var activeWriterId: UUID?
```

These are optional so existing snapshots without writers still decode correctly (backward compatible).

- [ ] **Step 4: Verify the project builds**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

Note: The new `Writer.swift` file must be added to the Xcode project's target. If using `xcodebuild`, you may need to add it via Xcode or by editing the `project.pbxproj` file to include the new source file in the compile sources build phase.

- [ ] **Step 5: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/Writer.swift Sources/SessionPersistence.swift
git commit -m "feat: add Writer data model and SessionWriterSnapshot

Writer represents a task/topic within a workspace. Each writer maps to a
t3code thread via t3codeThreadId. SessionWriterSnapshot extends persistence
for save/restore. Backward compatible with existing snapshots."
```

---

## Task 3: T3Code Sidecar Manager

**Why:** The sidecar manager handles the Node.js process lifecycle. Other components need it to get the server port for WebView URLs.

**Files:**
- Create: `cmux/Sources/T3CodeSidecarManager.swift`

- [ ] **Step 1: Create T3CodeSidecarManager.swift**

Create `cmux/Sources/T3CodeSidecarManager.swift`:

```swift
import Foundation
import os

/// Manages a t3code Node.js server process for a single workspace.
/// Spawns the server, parses the startup JSON to get the port,
/// monitors health, and handles restart/shutdown.
final class T3CodeSidecarManager {

    private let logger = Logger(subsystem: "com.cmuxterm.app", category: "T3CodeSidecar")

    /// The resolved server port after successful startup.
    private(set) var port: Int?

    /// The workspace's project directory (used as cwd and state-dir base).
    let projectDirectory: URL

    /// The running Node.js process.
    private var process: Process?

    /// Pipe for reading stdout to parse the startup signal.
    private var stdoutPipe: Pipe?

    /// Whether we're intentionally shutting down (suppress restart).
    private var isShuttingDown = false

    /// Callback when server becomes ready with its port.
    var onReady: ((Int) -> Void)?

    /// Callback when server crashes unexpectedly.
    var onCrash: (() -> Void)?

    init(projectDirectory: URL) {
        self.projectDirectory = projectDirectory
    }

    deinit {
        shutdown()
    }

    // MARK: - Lifecycle

    /// Start the t3code server process.
    /// Resolves the server binary, spawns the process, and parses stdout for the startup JSON signal.
    func start() {
        guard process == nil else {
            logger.warning("Sidecar already running for \(self.projectDirectory.path)")
            return
        }

        isShuttingDown = false

        let stateDir = projectDirectory.appendingPathComponent(".cmux").path

        // Create .cmux directory if needed
        try? FileManager.default.createDirectory(
            atPath: stateDir,
            withIntermediateDirectories: true
        )

        // Locate the t3code server binary
        guard let serverBinary = resolveServerBinary() else {
            logger.error("Could not find t3code server binary")
            return
        }

        let proc = Process()
        proc.executableURL = URL(fileURLWithPath: "/usr/bin/env")
        proc.arguments = [
            "node",
            serverBinary,
            "--state-dir", stateDir,
            "--port", "0",  // Let OS pick an available port
            "--auto-bootstrap-project-from-cwd",
            "--no-browser",
            "--mode", "web"
        ]
        proc.currentDirectoryURL = projectDirectory

        // Inherit environment but ensure PATH includes common Node locations
        var env = ProcessInfo.processInfo.environment
        let extraPaths = ["/usr/local/bin", "/opt/homebrew/bin"]
        if let existingPath = env["PATH"] {
            env["PATH"] = (extraPaths + [existingPath]).joined(separator: ":")
        }
        proc.environment = env

        let pipe = Pipe()
        proc.standardOutput = pipe
        proc.standardError = FileHandle.nullDevice
        self.stdoutPipe = pipe

        // Handle unexpected termination
        proc.terminationHandler = { [weak self] proc in
            guard let self = self, !self.isShuttingDown else { return }
            self.logger.error("t3code sidecar exited unexpectedly (code \(proc.terminationStatus))")
            self.process = nil
            self.port = nil
            DispatchQueue.main.async {
                self.onCrash?()
            }
        }

        do {
            try proc.run()
            self.process = proc
            logger.info("Spawned t3code sidecar (PID \(proc.processIdentifier)) for \(self.projectDirectory.path)")
        } catch {
            logger.error("Failed to spawn t3code sidecar: \(error.localizedDescription)")
            return
        }

        // Read stdout in background to parse startup JSON
        readStartupSignal(from: pipe)
    }

    /// Gracefully shut down the sidecar process.
    func shutdown() {
        isShuttingDown = true
        guard let proc = process, proc.isRunning else {
            process = nil
            return
        }

        logger.info("Shutting down t3code sidecar (PID \(proc.processIdentifier))")
        proc.terminate()  // SIGTERM

        // Force kill after 5 seconds if still running
        DispatchQueue.global().asyncAfter(deadline: .now() + 5) { [weak self] in
            guard let self = self, let proc = self.process, proc.isRunning else { return }
            self.logger.warning("Force killing t3code sidecar (PID \(proc.processIdentifier))")
            proc.interrupt()  // SIGINT as fallback
            DispatchQueue.global().asyncAfter(deadline: .now() + 2) {
                if proc.isRunning {
                    kill(proc.processIdentifier, SIGKILL)
                }
            }
        }

        process = nil
        port = nil
    }

    /// Restart the sidecar (used after crash detection).
    func restart() {
        shutdown()
        // Brief delay to let port be released
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) { [weak self] in
            self?.start()
        }
    }

    // MARK: - Private

    /// Parse the first JSON line from stdout to extract port info.
    private func readStartupSignal(from pipe: Pipe) {
        let handle = pipe.fileHandleForReading
        var buffer = Data()

        handle.readabilityHandler = { [weak self] fh in
            let data = fh.availableData
            guard !data.isEmpty else {
                // EOF
                fh.readabilityHandler = nil
                return
            }

            buffer.append(data)

            // Look for first complete JSON line
            if let newlineRange = buffer.range(of: Data("\n".utf8)) {
                let lineData = buffer.subdata(in: buffer.startIndex..<newlineRange.lowerBound)
                // Stop reading for startup signal after first line
                fh.readabilityHandler = nil

                self?.parseStartupSignal(lineData)
            }
        }
    }

    /// Parse the startup JSON and extract the port.
    private func parseStartupSignal(_ data: Data) {
        struct StartupSignal: Decodable {
            let type: String
            let port: Int
            let host: String?
        }

        do {
            let signal = try JSONDecoder().decode(StartupSignal.self, from: data)
            guard signal.type == "server-started" else {
                logger.error("Unexpected startup signal type: \(signal.type)")
                return
            }

            self.port = signal.port
            logger.info("t3code sidecar ready on port \(signal.port)")

            DispatchQueue.main.async { [weak self] in
                self?.onReady?(signal.port)
            }
        } catch {
            logger.error("Failed to parse t3code startup signal: \(error.localizedDescription)")
            if let str = String(data: data, encoding: .utf8) {
                logger.error("Raw stdout line: \(str)")
            }
        }
    }

    /// Find the t3code server binary.
    /// Checks: bundled path in app resources, then monorepo relative path.
    private func resolveServerBinary() -> String? {
        // Check for bundled server in app resources
        if let bundledPath = Bundle.main.path(forResource: "t3code-server", ofType: "mjs") {
            return bundledPath
        }

        // Check monorepo relative path (development mode)
        // Assumes cmux and t3code are siblings under the same parent directory
        let monorepoPath = projectDirectory
            .deletingLastPathComponent()  // Go up from project dir — won't work in general
        // Instead, use a fixed known path or environment variable
        if let t3codePath = ProcessInfo.processInfo.environment["T3CODE_SERVER_PATH"] {
            if FileManager.default.fileExists(atPath: t3codePath) {
                return t3codePath
            }
        }

        // Fallback: try to find via PATH (npx t3 approach)
        // In production, the server will be bundled. For development,
        // set T3CODE_SERVER_PATH environment variable.
        let knownPaths = [
            "/usr/local/lib/node_modules/t3/dist/index.mjs",
            "/opt/homebrew/lib/node_modules/t3/dist/index.mjs",
        ]
        for path in knownPaths {
            if FileManager.default.fileExists(atPath: path) {
                return path
            }
        }

        logger.warning("t3code server binary not found. Set T3CODE_SERVER_PATH env var.")
        return nil
    }
}
```

- [ ] **Step 2: Add file to Xcode project and verify build**

Add `T3CodeSidecarManager.swift` to the Xcode project target compile sources. Then build:

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 3: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/T3CodeSidecarManager.swift
git commit -m "feat: add T3CodeSidecarManager for sidecar lifecycle

Spawns one t3code Node.js server per workspace, parses startup JSON
from stdout to get port, monitors health, and handles restart/shutdown.
Uses SIGTERM with 5s grace period before SIGKILL."
```

---

## Task 4: ChatPanel (WebView wrapper for t3code UI)

**Why:** Each writer needs a panel type that displays the t3code chat UI. This follows the existing Panel protocol used by terminal and browser panels.

**Files:**
- Create: `cmux/Sources/Panels/ChatPanel.swift`
- Modify: `cmux/Sources/SessionPersistence.swift` (add `.chat` to PanelType if it's an enum)
- Reference: `cmux/Sources/Panels/BrowserPanel.swift` (pattern to follow)

- [ ] **Step 1: Identify the Panel protocol and PanelType**

Read `cmux/Sources/Workspace.swift` and `cmux/Sources/Panels/BrowserPanel.swift` to understand the exact `Panel` protocol conformance and `PanelType` enum. The ChatPanel must conform to the same protocol.

Key points from exploration:
- `BrowserPanel: Panel, ObservableObject` (line 1606 of BrowserPanel.swift)
- Panel has: `id: UUID`, `panelType: PanelType`
- PanelType has cases: `.terminal`, `.browser`, `.markdown`
- Workspace stores panels in `var panels: [UUID: any Panel]`

- [ ] **Step 2: Add `.chat` case to PanelType**

Find `PanelType` enum (likely in a Panel protocol file or SessionPersistence.swift). Add:

```swift
case chat
```

Also add the corresponding Codable support if PanelType is Codable.

- [ ] **Step 3: Create ChatPanel.swift**

Create `cmux/Sources/Panels/ChatPanel.swift`:

```swift
import Foundation
import WebKit
import Combine
import os

/// A panel that displays the t3code chat UI in a CmuxWebView.
/// Each ChatPanel corresponds to one writer/task.
final class ChatPanel: Panel, ObservableObject {

    private let logger = Logger(subsystem: "com.cmuxterm.app", category: "ChatPanel")

    let id: UUID
    let panelType: PanelType = .chat

    /// The workspace this panel belongs to.
    private(set) var workspaceId: UUID

    /// The writer's t3code thread ID (set after first conversation).
    @Published var t3codeThreadId: String?

    /// The web view displaying t3code's React UI.
    private(set) var webView: CmuxWebView

    /// Whether the WebView should be rendered (always true for chat).
    @Published var shouldRenderWebView: Bool = true

    /// The t3code server port for this workspace's sidecar.
    private var serverPort: Int?

    init(workspaceId: UUID, threadId: String? = nil, serverPort: Int? = nil) {
        self.id = UUID()
        self.workspaceId = workspaceId
        self.t3codeThreadId = threadId
        self.serverPort = serverPort

        // Create WKWebView configuration
        let config = WKWebViewConfiguration()
        config.preferences.setValue(true, forKey: "developerExtrasEnabled")

        // Allow local network access for localhost connections
        config.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs")

        self.webView = CmuxWebView(frame: .zero, configuration: config)
        self.webView.allowsBackForwardNavigationGestures = false

        // Load the t3code UI if we have a port
        if let port = serverPort {
            loadT3CodeUI(port: port)
        }
    }

    /// Load or reload the t3code UI with the given server port.
    func loadT3CodeUI(port: Int) {
        self.serverPort = port

        var urlString = "http://localhost:\(port)"
        if let threadId = t3codeThreadId {
            urlString += "/#/\(threadId)"
        }

        guard let url = URL(string: urlString) else {
            logger.error("Invalid t3code URL: \(urlString)")
            return
        }

        logger.info("Loading t3code UI: \(urlString)")
        webView.load(URLRequest(url: url))
    }

    /// Update the thread ID (called when user starts a new conversation).
    func setThreadId(_ threadId: String) {
        self.t3codeThreadId = threadId
    }

    /// Reload the web view (used after sidecar restart).
    func reload() {
        if let port = serverPort {
            loadT3CodeUI(port: port)
        }
    }
}
```

- [ ] **Step 4: Verify build**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

Note: You will need to conform to all required properties/methods of the `Panel` protocol. Read the protocol definition first and add any missing conformance. The above is a starting point — the exact protocol may require additional properties like `title`, `directory`, etc.

- [ ] **Step 5: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/Panels/ChatPanel.swift Sources/SessionPersistence.swift
git commit -m "feat: add ChatPanel for t3code WebView integration

ChatPanel wraps CmuxWebView to display t3code's React UI. Follows the
same Panel protocol as BrowserPanel and TerminalPanel. Supports thread
ID routing and sidecar port updates for reconnection."
```

---

## Task 5: Writer Sidebar View

**Why:** The sidebar needs to change from a flat list of workspace tabs to a two-level tree: workspaces (collapsible) containing writers.

**Files:**
- Create: `cmux/Sources/WriterSidebarView.swift`
- Modify: `cmux/Sources/ContentView.swift:7946-8042` (VerticalTabsSidebar body)

- [ ] **Step 1: Create WriterSidebarView.swift**

Create `cmux/Sources/WriterSidebarView.swift`. This is a SwiftUI view that renders the workspace → writer hierarchy:

```swift
import SwiftUI

/// A single writer row in the sidebar, nested under a workspace.
struct WriterSidebarRow: View {
    @ObservedObject var writer: Writer
    let isSelected: Bool
    let onSelect: () -> Void
    let onRename: (String) -> Void
    let onDelete: () -> Void

    @State private var isEditing = false
    @State private var editName = ""

    var body: some View {
        HStack(spacing: 8) {
            // Writer icon
            Text("\u{270E}")  // ✎
                .font(.system(size: 11))
                .foregroundColor(.secondary)

            if isEditing {
                TextField("Writer name", text: $editName, onCommit: {
                    onRename(editName)
                    isEditing = false
                })
                .textFieldStyle(.plain)
                .font(.system(size: 12))
            } else {
                Text(writer.name)
                    .font(.system(size: 12))
                    .foregroundColor(isSelected ? .primary : .secondary)
                    .lineLimit(1)
            }

            Spacer()

            // Active indicator
            if writer.isActive {
                Circle()
                    .fill(Color.green)
                    .frame(width: 6, height: 6)
            }
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 6)
        .padding(.leading, 12)  // Extra indent for nesting under workspace
        .background(
            isSelected
                ? Color.accentColor.opacity(0.1)
                : Color.clear
        )
        .contentShape(Rectangle())
        .onTapGesture { onSelect() }
        .contextMenu {
            Button("Rename") {
                editName = writer.name
                isEditing = true
            }
            Divider()
            Button("Delete", role: .destructive) { onDelete() }
        }
    }
}

/// The writers list for a single workspace (shown when workspace is expanded).
struct WorkspaceWritersList: View {
    let writers: [Writer]
    let selectedWriterId: UUID?
    let onSelectWriter: (Writer) -> Void
    let onNewWriter: () -> Void
    let onRenameWriter: (Writer, String) -> Void
    let onDeleteWriter: (Writer) -> Void

    var body: some View {
        VStack(spacing: 0) {
            ForEach(writers) { writer in
                WriterSidebarRow(
                    writer: writer,
                    isSelected: writer.id == selectedWriterId,
                    onSelect: { onSelectWriter(writer) },
                    onRename: { newName in onRenameWriter(writer, newName) },
                    onDelete: { onDeleteWriter(writer) }
                )
            }

            // "+ New writer" button
            Button(action: onNewWriter) {
                HStack(spacing: 8) {
                    Image(systemName: "plus")
                        .font(.system(size: 12))
                    Text("New writer")
                        .font(.system(size: 11))
                }
                .foregroundColor(.secondary.opacity(0.6))
                .padding(.horizontal, 12)
                .padding(.vertical, 6)
                .padding(.leading, 12)
            }
            .buttonStyle(.plain)
        }
    }
}
```

- [ ] **Step 2: Integrate into VerticalTabsSidebar**

In `cmux/Sources/ContentView.swift`, find the `VerticalTabsSidebar` body (lines 7946-8042). Inside the `LazyVStack` where `TabItemView` is rendered for each workspace, add the writer list below each tab when the workspace is expanded.

The integration depends on the existing sidebar structure. The key change is:

After each `TabItemView(tab: tab, ...)`, conditionally render the `WorkspaceWritersList` if the workspace is expanded:

```swift
// Inside the ForEach that renders TabItemView:
TabItemView(tab: tab, /* existing params */)

// Add below TabItemView:
if tab.isWritersExpanded {
    WorkspaceWritersList(
        writers: tab.writers,
        selectedWriterId: tab.activeWriterId,
        onSelectWriter: { writer in
            tabManager.selectWriter(writer, inWorkspace: tab)
        },
        onNewWriter: {
            tabManager.createWriter(inWorkspace: tab)
        },
        onRenameWriter: { writer, name in
            writer.name = name
        },
        onDeleteWriter: { writer in
            tabManager.deleteWriter(writer, fromWorkspace: tab)
        }
    )
}
```

Note: `Tab` (the workspace model) will need new properties: `writers: [Writer]`, `isWritersExpanded: Bool`, `activeWriterId: UUID?`. Add these in a subsequent step after understanding the Tab class structure.

- [ ] **Step 3: Verify build**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 4: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/WriterSidebarView.swift Sources/ContentView.swift
git commit -m "feat: add writer sidebar with workspace → writer hierarchy

Two-level collapsible sidebar: workspaces expand to show nested writers.
Each writer row supports select, rename, and delete. New writer button
at bottom of each workspace's list."
```

---

## Task 6: Writer Content View (Chat + Terminal Splits)

**Why:** When a writer is selected, the main area needs to show the chat WebView as the root pane with optional terminal splits.

**Files:**
- Create: `cmux/Sources/WriterContentView.swift`
- Modify: `cmux/Sources/Workspace.swift` (extend `createPanel` and split logic)

- [ ] **Step 1: Create WriterContentView.swift**

Create `cmux/Sources/WriterContentView.swift`. This is the container that hosts the chat WebView as the root pane and manages terminal splits:

```swift
import SwiftUI
import WebKit

/// The main content area for a writer.
/// Shows the t3code chat WebView as the root, with optional Ghostty terminal splits.
struct WriterContentView: View {
    @ObservedObject var writer: Writer
    let workspace: Workspace  // Access to split infrastructure
    let chatPanel: ChatPanel

    var body: some View {
        // The actual rendering is handled by Bonsplit via the workspace's
        // existing pane/split infrastructure. This view manages the writer-level
        // concerns: ensuring a chat panel exists and connecting it to the workspace.
        //
        // The chat panel's WKWebView is hosted in the Bonsplit pane tree
        // alongside terminal panels. The workspace's existing NSView hierarchy
        // handles the actual layout.
        //
        // For the SwiftUI integration, we wrap the chat panel's web view:
        WriterContentViewRepresentable(chatPanel: chatPanel)
    }
}

/// NSViewRepresentable wrapper for the chat panel's WKWebView.
/// Used to embed the WKWebView in SwiftUI context.
struct WriterContentViewRepresentable: NSViewRepresentable {
    let chatPanel: ChatPanel

    func makeNSView(context: Context) -> CmuxWebView {
        return chatPanel.webView
    }

    func updateNSView(_ nsView: CmuxWebView, context: Context) {
        // WebView updates handled by ChatPanel directly
    }
}
```

Note: The actual split management is done via Bonsplit in the Workspace class. This view provides the SwiftUI bridge. The Bonsplit controller manages the pane tree (chat pane as root, terminal panes as splits).

- [ ] **Step 2: Extend Workspace to support chat panels in splits**

In `cmux/Sources/Workspace.swift`, find the `createPanel(from:inPane:)` method (lines 534-581). Add a case for `.chat`:

```swift
case .chat:
    // Chat panels are created via newChatSurface, not from snapshots during normal flow.
    // During session restore, recreate the chat panel for the writer.
    if let threadId = snapshot.t3codeThreadId {
        let panel = newChatSurface(
            inPane: paneId,
            threadId: threadId,
            focus: false
        )
    }
```

Also add a `newChatSurface` method to Workspace:

```swift
/// Create a new chat panel in the given pane.
@discardableResult
func newChatSurface(
    inPane paneId: PaneID,
    threadId: String? = nil,
    focus: Bool = true
) -> ChatPanel? {
    guard let sidecarManager = self.t3codeSidecarManager else { return nil }

    let panel = ChatPanel(
        workspaceId: self.id,
        threadId: threadId,
        serverPort: sidecarManager.port
    )

    panels[panel.id] = panel
    bonsplitController.addTab(panel.id, inPane: paneId)

    if focus {
        bonsplitController.selectTab(panel.id, inPane: paneId)
        bonsplitController.focusPane(paneId)
    }

    return panel
}
```

Note: Workspace will need a `t3codeSidecarManager: T3CodeSidecarManager?` property, set by the AppDelegate when the workspace is created.

- [ ] **Step 3: Extend SessionPanelSnapshot for chat panels**

In `cmux/Sources/SessionPersistence.swift`, extend `SessionPanelSnapshot` (lines 243-257) to include chat-specific data:

```swift
// Add to SessionPanelSnapshot properties:
var t3codeThreadId: String?  // Only set for .chat panels
```

- [ ] **Step 4: Verify build**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 5: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/WriterContentView.swift Sources/Workspace.swift Sources/SessionPersistence.swift
git commit -m "feat: add WriterContentView with chat panel integration

WriterContentView hosts the t3code chat WebView as the root pane.
Extends Workspace with newChatSurface() and chat panel creation during
session restore. Chat panels coexist with terminal splits via Bonsplit."
```

---

## Task 7: Integrate Sidecar into App Lifecycle

**Why:** The sidecar needs to start with the app and shut down cleanly. Workspaces need to be connected to their sidecar managers.

**Files:**
- Modify: `cmux/Sources/AppDelegate.swift` (startup, shutdown, workspace creation)
- Modify: `cmux/Sources/Workspace.swift` (add sidecar manager property)

- [ ] **Step 1: Add sidecar manager property to Workspace**

In `cmux/Sources/Workspace.swift`, add near the other properties (around line 4790):

```swift
/// The t3code sidecar manager for this workspace. Set by AppDelegate.
var t3codeSidecarManager: T3CodeSidecarManager?
```

- [ ] **Step 2: Add sidecar management to AppDelegate**

In `cmux/Sources/AppDelegate.swift`, add a dictionary to track sidecar managers:

```swift
// Add near other private properties (around line 2070):
private var sidecarManagers: [UUID: T3CodeSidecarManager] = [:]
```

- [ ] **Step 3: Start sidecars when workspaces are created**

Find where workspaces are initialized in AppDelegate (likely in the session restore flow or new workspace creation). Add sidecar startup:

```swift
/// Start a t3code sidecar for the given workspace.
private func startSidecar(for workspace: Workspace) {
    guard let directory = workspace.directory else { return }

    let manager = T3CodeSidecarManager(projectDirectory: directory)

    manager.onReady = { [weak workspace] port in
        // Notify all chat panels in this workspace of the port
        workspace?.notifyChatPanelsOfPort(port)
    }

    manager.onCrash = { [weak self, weak workspace] in
        guard let self = self, let workspace = workspace else { return }
        // Auto-restart after crash
        self.sidecarManagers[workspace.id]?.restart()
    }

    sidecarManagers[workspace.id] = manager
    workspace.t3codeSidecarManager = manager
    manager.start()
}
```

- [ ] **Step 4: Shut down all sidecars on app quit**

Find `applicationWillTerminate` or the app shutdown path in AppDelegate. Add:

```swift
// In the app termination flow:
for (_, manager) in sidecarManagers {
    manager.shutdown()
}
sidecarManagers.removeAll()
```

- [ ] **Step 5: Add notifyChatPanelsOfPort to Workspace**

In `cmux/Sources/Workspace.swift`, add a method to update chat panels when the sidecar reports its port:

```swift
/// Called when the t3code sidecar becomes ready. Updates all chat panels with the port.
func notifyChatPanelsOfPort(_ port: Int) {
    for (_, panel) in panels {
        if let chatPanel = panel as? ChatPanel {
            chatPanel.loadT3CodeUI(port: port)
        }
    }
}
```

- [ ] **Step 6: Verify build**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 7: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/AppDelegate.swift Sources/Workspace.swift
git commit -m "feat: integrate sidecar lifecycle into app startup/shutdown

AppDelegate starts a T3CodeSidecarManager per workspace on launch and
shuts them all down on app quit. Workspaces notify chat panels when
their sidecar becomes ready. Auto-restart on crash."
```

---

## Task 8: Writer CRUD Operations

**Why:** Users need to create, rename, reorder, and delete writers within workspaces.

**Files:**
- Modify: `cmux/Sources/Workspace.swift` or the Tab/TabManager class (wherever workspace state is managed)

- [ ] **Step 1: Identify the Tab/TabManager class**

Read the TabManager class to understand how workspaces (tabs) are managed. The writer CRUD operations need to live where workspace state is managed. Key questions:
- Where is `Tab` defined?
- How does `TabManager` manage the list of tabs?
- Where should writer state live — on Tab or Workspace?

- [ ] **Step 2: Add writer management methods**

Add to the appropriate class (likely Tab or Workspace):

```swift
// MARK: - Writer Management

/// All writers in this workspace.
@Published var writers: [Writer] = []

/// The currently active writer.
@Published var activeWriterId: UUID?

/// Whether the writer list is expanded in the sidebar.
@Published var isWritersExpanded: Bool = true

/// Create a new writer with a default name.
func createWriter(name: String? = nil) -> Writer {
    let writerName = name ?? "New task \(writers.count + 1)"
    let writer = Writer(name: writerName)
    writers.append(writer)
    activeWriterId = writer.id
    return writer
}

/// Delete a writer and clean up its resources.
func deleteWriter(_ writer: Writer) {
    writers.removeAll { $0.id == writer.id }
    // TODO: Clean up associated chat panel and terminal splits
    if activeWriterId == writer.id {
        activeWriterId = writers.first?.id
    }
}

/// Rename a writer.
func renameWriter(_ writer: Writer, to name: String) {
    writer.name = name
}

/// Move a writer from one index to another (reorder).
func moveWriter(from source: IndexSet, to destination: Int) {
    writers.move(fromOffsets: source, toOffset: destination)
}
```

- [ ] **Step 3: Verify build**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 4: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/Workspace.swift
git commit -m "feat: add writer CRUD operations

Create, rename, reorder, and delete writers within a workspace.
Active writer tracking for sidebar selection."
```

---

## Task 9: Session Persistence for Writers

**Why:** Writers and their state need to survive app restarts.

**Files:**
- Modify: `cmux/Sources/SessionPersistence.swift`
- Modify: `cmux/Sources/Workspace.swift` (snapshot building and restoration)
- Modify: `cmux/Sources/AppDelegate.swift` (snapshot building)

- [ ] **Step 1: Extend snapshot building to include writers**

Find where `SessionWorkspaceSnapshot` is built (in AppDelegate's `buildSessionSnapshot` or in Workspace). Add writer snapshot creation:

```swift
// When building SessionWorkspaceSnapshot:
let writerSnapshots = workspace.writers.map { writer in
    SessionWriterSnapshot(
        id: writer.id,
        name: writer.name,
        t3codeThreadId: writer.t3codeThreadId,
        layout: nil  // TODO: capture per-writer split layout if needed
    )
}
// Set on the workspace snapshot:
workspaceSnapshot.writers = writerSnapshots
workspaceSnapshot.activeWriterId = workspace.activeWriterId
```

- [ ] **Step 2: Extend session restore to recreate writers**

In `Workspace.restoreSessionSnapshot()` (line 214), add writer restoration:

```swift
// After restoring layout, restore writers:
if let writerSnapshots = snapshot.writers {
    self.writers = writerSnapshots.map { ws in
        Writer(
            id: ws.id,
            name: ws.name,
            t3codeThreadId: ws.t3codeThreadId
        )
    }
    self.activeWriterId = snapshot.activeWriterId ?? writerSnapshots.first?.id
    self.isWritersExpanded = true
}
```

- [ ] **Step 3: Verify build**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 4: Test persistence round-trip**

1. Launch cmux (debug build)
2. Create a workspace with a project directory
3. Create 2-3 writers, give them names
4. Quit cmux
5. Relaunch cmux
6. Verify writers are restored with correct names and order

- [ ] **Step 5: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add Sources/SessionPersistence.swift Sources/Workspace.swift Sources/AppDelegate.swift
git commit -m "feat: persist writers across app restarts

Writers are saved to SessionWorkspaceSnapshot and restored on launch.
Thread IDs preserved for t3code session recovery. Backward compatible
with snapshots that don't have writers."
```

---

## Task 10: End-to-End Integration & Wiring

**Why:** All the pieces exist — now wire them together so the full flow works.

**Files:**
- Modify: Multiple files for final wiring

- [ ] **Step 1: Wire writer selection to content view switching**

When a writer is selected in the sidebar, the main content area must switch to show that writer's chat panel + terminal splits. Find where the main content view is rendered (in ContentView.swift) and add the switching logic:

```swift
// When activeWriterId changes, update the displayed pane set in Bonsplit
// to show the chat panel (and terminal splits) associated with the active writer.
```

This requires understanding how the current tab selection switches the displayed workspace content. Follow the same pattern but at the writer level within a workspace.

- [ ] **Step 2: Wire "New writer" to create chat panel**

When `createWriter()` is called, it should also create a ChatPanel in the workspace:

```swift
func createWriter(name: String? = nil) -> Writer {
    let writer = Writer(name: name ?? "New task \(writers.count + 1)")
    writers.append(writer)
    activeWriterId = writer.id

    // Create a chat panel for this writer
    if let sidecarManager = t3codeSidecarManager,
       let rootPaneId = bonsplitController.allPaneIds.first {
        let chatPanel = newChatSurface(
            inPane: rootPaneId,
            threadId: nil,
            focus: true
        )
        // Associate chat panel with writer
        writer.chatPanelId = chatPanel?.id
    }

    return writer
}
```

Note: Writer will need a `chatPanelId: UUID?` property to track which chat panel belongs to it.

- [ ] **Step 3: Wire Cmd+D to split terminal within writer context**

The existing Cmd+D split should create a Ghostty terminal pane (not a chat pane) when triggered within a writer context. The `splitTabBar(_:didSplitPane:newPane:orientation:)` delegate (Workspace.swift line 9335) already auto-creates terminals in new panes. Verify this works correctly when the root pane is a chat panel.

- [ ] **Step 4: Wire sidecar port to existing chat panels on startup**

During session restore, after the sidecar starts and reports its port, all restored chat panels need to be notified. This is already handled by `notifyChatPanelsOfPort()` via the `onReady` callback. Verify the timing is correct — sidecar start must happen after panels are restored.

- [ ] **Step 5: Manual end-to-end test**

Test the full flow:
1. Launch cmux
2. Create a new workspace pointing to a project directory with a Node.js project
3. Verify the t3code sidecar starts (check Activity Monitor for node process)
4. Create a new writer — verify the chat WebView loads
5. Type a message in the chat — verify Codex responds
6. Press Cmd+D — verify a Ghostty terminal appears below the chat
7. The terminal should be in the project directory
8. Create a second writer — verify it gets its own chat
9. Switch between writers — verify the content switches
10. Quit and relaunch — verify writers and chats are restored

- [ ] **Step 6: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add -A
git commit -m "feat: wire end-to-end writer + chat + terminal integration

Complete integration: writer selection switches content view, new writer
creates chat panel, Cmd+D splits terminal within writer, sidecar port
propagated to chat panels on startup and after crashes."
```

---

## Task 11: Polish & Edge Cases

**Why:** Handle error states, edge cases, and UX polish for production readiness.

**Files:**
- Various files for fixes and polish

- [ ] **Step 1: Handle sidecar startup failure gracefully**

If Node.js isn't installed or the server binary isn't found, show a helpful error in the chat panel area instead of a blank WebView:

```swift
// In ChatPanel, if server port is nil, show error state
// Could be a simple HTML page loaded from a string:
let errorHTML = """
<html><body style="background:#1a1a1a;color:#aaa;font-family:system-ui;
display:flex;align-items:center;justify-content:center;height:100vh;margin:0;">
<div style="text-align:center">
<h2>t3code server not available</h2>
<p>Waiting for server to start...</p>
</div></body></html>
"""
webView.loadHTMLString(errorHTML, baseURL: nil)
```

- [ ] **Step 2: Handle WebSocket disconnection in WebView**

The t3code React UI already handles WebSocket disconnection with a "Reconnecting..." state. Verify this works when the sidecar crashes and restarts. The WebView should automatically reconnect once the sidecar is back.

- [ ] **Step 3: Prevent closing the chat panel**

Override the close behavior for chat panels so they can't be closed via Cmd+W when focused. Only terminal splits can be closed. The chat is the writer's root and persists until the writer is deleted.

```swift
// In Workspace's panel close logic:
if let panel = panels[panelId], panel.panelType == .chat {
    // Don't close — chat is the writer's root
    return
}
```

- [ ] **Step 4: Default writer on workspace creation**

When a new workspace is created, automatically create one default writer named after the project directory:

```swift
// After workspace creation:
let defaultWriter = workspace.createWriter(
    name: projectDirectory.lastPathComponent
)
```

- [ ] **Step 5: Theme sync between cmux and t3code WebView**

Inject the cmux theme colors into the WebView so the chat UI blends visually:

```swift
// After WebView loads, inject CSS overrides:
let css = "body { background-color: \(cmuxBackgroundColor); }"
let js = "document.head.insertAdjacentHTML('beforeend', '<style>\(css)</style>');"
webView.evaluateJavaScript(js)
```

This is a basic approach. For deeper theme integration, the t3code React UI would need to accept theme parameters.

- [ ] **Step 6: Add .cmux to .gitignore**

The `.cmux/` directory in project roots (containing `state.sqlite`) should be gitignored:

```bash
echo ".cmux/" >> /Users/timokruth/Projekte/cmux-t3code/.gitignore
```

- [ ] **Step 7: Commit**

```bash
cd /Users/timokruth/Projekte/cmux-t3code/cmux
git add -A
git commit -m "feat: polish and edge case handling

Graceful sidecar failure, WebSocket reconnection, chat panel close
protection, default writer on workspace creation, basic theme sync,
and .cmux gitignore."
```

---

## Summary

| Task | Component | Estimated Complexity |
|------|-----------|---------------------|
| 1 | t3code startup signal | Small (TypeScript) |
| 2 | Writer data model | Small (Swift) |
| 3 | Sidecar manager | Medium (Swift) |
| 4 | ChatPanel | Medium (Swift) |
| 5 | Writer sidebar view | Medium (SwiftUI) |
| 6 | Writer content view | Medium (Swift/SwiftUI) |
| 7 | App lifecycle integration | Medium (Swift) |
| 8 | Writer CRUD | Small (Swift) |
| 9 | Session persistence | Medium (Swift) |
| 10 | End-to-end wiring | Large (Swift) |
| 11 | Polish & edge cases | Medium (Mixed) |

**Dependencies:** Task 1 (t3code) is independent. Tasks 2-4 can be done in parallel. Tasks 5-6 depend on 2 and 4. Task 7 depends on 3. Tasks 8-9 depend on 2. Task 10 depends on all prior tasks. Task 11 depends on 10.

```
Task 1 (t3code) ──────────────────────────────────────┐
Task 2 (Writer model) ──┬── Task 5 (Sidebar) ─────────┤
Task 3 (Sidecar) ───────┼── Task 7 (App lifecycle) ───┤
Task 4 (ChatPanel) ─────┴── Task 6 (Content view) ────┼── Task 10 (Wiring) ── Task 11 (Polish)
Task 2 ──────────────────── Task 8 (CRUD) ─────────────┤
Task 2 ──────────────────── Task 9 (Persistence) ──────┘
```
