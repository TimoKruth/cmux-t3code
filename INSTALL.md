# cmux-t3code Installation Guide

Two ways to run cmux with t3code: **bundled** (all-in-one app) or **standalone** (separate processes).

## Prerequisites

- macOS 14+
- Xcode 16+ (with command-line tools)
- [Zig](https://ziglang.org/) (`brew install zig`)
- [Bun](https://bun.sh/) (`brew install oven-sh/bun/bun`) or Node.js 22+

## Approach 1: Bundled (recommended)

The t3code server is built and embedded inside cmux.app. Every project opened in cmux automatically gets its own t3code sidecar — no external setup needed.

### Quick install

```bash
git clone --recursive <repo-url> cmux-t3code
cd cmux-t3code

# 1. Setup cmux (builds GhosttyKit, etc.)
cd cmux && ./scripts/setup.sh && cd ..

# 2. Build & install everything (t3code + cmux → /Applications/cmux-t3code.app)
cd cmux && ./scripts/install-cmux-t3code.sh
```

### What the install script does

1. Builds the t3code server (`bun install && bun run build` in `t3code/`)
2. Builds cmux in Release mode — the Xcode build phase automatically copies `t3code/apps/server/dist/` into `cmux.app/Contents/Resources/t3code-server/`
3. Builds cmuxd (local daemon) and the Ghostty CLI helper
4. Installs to `/Applications/cmux-t3code.app` with a unique bundle ID
5. Symlinks the CLI to your PATH

### Manual step-by-step

If you prefer to run each step yourself:

```bash
cd cmux-t3code

# Build t3code server
cd t3code
bun install
bun run build
cd ..

# Verify the dist was created
ls t3code/apps/server/dist/index.mjs

# Setup cmux (first time only)
cd cmux
./scripts/setup.sh

# Build and install
./scripts/install-cmux-t3code.sh
```

### Rebuilding after changes

After pulling new changes or modifying t3code:

```bash
cd cmux-t3code/t3code && bun run build && cd ..
cd cmux && ./scripts/install-cmux-t3code.sh
```

For development/debug builds:

```bash
cd cmux-t3code/t3code && bun run build && cd ..
cd cmux && ./scripts/reload.sh --tag dev
```

## Approach 2: Standalone (separate t3code server)

Run the t3code server independently and point cmux at it. Useful for development or when you want to manage the server lifecycle yourself.

### 1. Start the t3code server

```bash
cd cmux-t3code/t3code

# Install dependencies (first time)
bun install

# Start the dev server (with hot reload)
bun run dev:server

# Or build and run the production server
bun run build
node apps/server/dist/index.mjs \
  --port 3773 \
  --home-dir /path/to/project/.cmux \
  --auto-bootstrap-project-from-cwd \
  --no-browser \
  --mode web
```

The server will print the port it's listening on.

### 2. Point cmux at the running server

Set the `T3CODE_SERVER_PATH` environment variable before launching cmux to override the bundled/discovered binary:

```bash
export T3CODE_SERVER_PATH="$HOME/Projekte/cmux-t3code/t3code/apps/server/dist/index.mjs"
open /Applications/cmux-t3code.app
```

Or set it permanently via `launchctl` (persists across app launches):

```bash
launchctl setenv T3CODE_SERVER_PATH "$HOME/Projekte/cmux-t3code/t3code/apps/server/dist/index.mjs"
```

When `T3CODE_SERVER_PATH` is set, cmux will use that binary instead of the bundled one. Each project still gets its own sidecar process managed by cmux.

### 3. For fully external server management

If you want to run the t3code server entirely outside of cmux (e.g., for debugging the server itself), you can start it manually per-project:

```bash
cd /path/to/your/project

node ~/Projekte/cmux-t3code/t3code/apps/server/dist/index.mjs \
  --port 4000 \
  --home-dir .cmux \
  --auto-bootstrap-project-from-cwd \
  --no-browser \
  --mode web
```

Then open `http://localhost:4000` in a browser to access the t3code UI directly.

## Server binary resolution order

When cmux starts a t3code sidecar for a project, it looks for the server binary in this order:

1. `T3CODE_SERVER_PATH` environment variable
2. (Debug only) `CMUXSourceRoot` from Info.plist → `../t3code/apps/server/dist/index.mjs`
3. Walk up from the project directory looking for a `t3code/` sibling (stops at `.git` boundary)
4. (Debug only) Well-known dev paths (`~/Projekte/cmux-t3code/...`, `~/Projects/cmux-t3code/...`)
5. Global install paths (`/usr/local/lib/node_modules/t3/dist/index.mjs`, `/opt/homebrew/...`)
6. App bundle: `cmux.app/Contents/Resources/t3code-server/index.mjs`

## Troubleshooting

**"Starting t3code server..." hangs forever**

The t3code server binary wasn't found. Check:
- Was `bun run build` run in `t3code/`? (`ls t3code/apps/server/dist/index.mjs`)
- Was cmux rebuilt after building t3code? (The Xcode build phase copies the dist)
- Is Node.js available? (`which node`)

**Server crashes on startup**

Check the sidecar logs:
```bash
cat /path/to/project/.cmux/logs/server.log
```

Check system logs:
```bash
log show --predicate 'process == "cmux"' --last 10m | grep -i t3code
```

**Database conflicts between cmux-t3code and standalone T3 Code**

cmux-t3code stores state per-project in `{project}/.cmux/state.sqlite`. Standalone T3 Code uses `~/.t3/userdata/state.sqlite`. These are isolated by design. Do not point one at the other's state directory.
