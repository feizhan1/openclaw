# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenClaw is a personal AI assistant that runs on your own devices. It connects to multiple messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, etc.) through a Gateway daemon and provides AI-powered responses using various LLM providers (Anthropic, OpenAI, etc.).

**Repository:** https://github.com/openclaw/openclaw

## Core Architecture

### Gateway-Centric Design
- **Gateway daemon** (`src/gateway/`) is the control plane that owns all messaging surface connections
- Runs as a long-lived WebSocket server (default `127.0.0.1:18789`)
- All clients (macOS app, CLI, web UI, mobile apps) connect to the Gateway via WebSocket
- Protocol is defined using TypeBox schemas (`src/gateway/protocol/`)
- Canvas host runs separately on port `18793` for agent-editable HTML/A2UI

### Agent System
- **Pi Agent Core** (`src/agents/`) handles LLM interactions and tool execution
- Agent sessions stored in `~/.clawdbot/agents/<agentId>/sessions/*.jsonl`
- Supports multiple LLM providers with OAuth and API key authentication
- Tool definitions in `src/agents/tools/`

### Channel Architecture
- **Core channels:** `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web` (WhatsApp via Baileys), `src/channels`, `src/routing`
- **Extension channels:** `extensions/*` (e.g., `extensions/msteams`, `extensions/matrix`, `extensions/zalo`)
- When refactoring shared logic, always consider ALL built-in + extension channels
- Channel routing and pairing logic in `src/routing/` and `src/pairing/`

### Plugin System
- Plugins live in `extensions/*` as workspace packages
- Runtime dependencies must be in `dependencies`, NOT `devDependencies`
- Avoid `workspace:*` in `dependencies` (breaks npm install)
- Put `openclaw` in `devDependencies` or `peerDependencies`
- Plugin SDK exported at `openclaw/plugin-sdk`

### Mobile Apps
- **macOS:** `apps/macos/` (SwiftUI, uses Observation framework)
- **iOS:** `apps/ios/` (SwiftUI, uses XcodeGen)
- **Android:** `apps/android/` (Kotlin)
- Mac app IS the gateway on macOS (no separate LaunchAgent)

## Development Commands

### Setup
```bash
pnpm install              # Install dependencies
pnpm ui:build            # Build web UI (auto-installs deps)
pnpm build               # TypeScript compilation (tsc)
```

### Development
```bash
pnpm openclaw [args]     # Run CLI in dev mode (uses Bun)
pnpm dev                 # Alternative dev runner
pnpm gateway:watch       # Auto-reload gateway on changes
pnpm gateway:dev         # Gateway with channel skip flags
```

### Testing
```bash
pnpm test                      # Run all tests (Vitest)
pnpm test:coverage            # Run with coverage report
pnpm test:watch               # Watch mode
pnpm test:live                # Live tests (requires CLAWDBOT_LIVE_TEST=1)
pnpm test:e2e                 # E2E tests
pnpm test:docker:onboard      # Docker onboarding E2E
pnpm test:docker:live-models  # Docker live model tests
```

**Testing notes:**
- Tests colocated as `*.test.ts`, E2E as `*.e2e.test.ts`
- Coverage thresholds: 70% lines/functions/statements, 55% branches
- Max 16 workers (already tried higher, don't change)
- Check for connected real devices (iOS/Android) before using simulators

### Linting & Formatting
```bash
pnpm lint           # Oxlint (TypeScript)
pnpm lint:swift     # SwiftLint
pnpm format         # Check formatting with Oxfmt
pnpm format:fix     # Auto-fix formatting
```

### Mobile Apps
```bash
# iOS
pnpm ios:gen        # Generate Xcode project
pnpm ios:open       # Open in Xcode
pnpm ios:build      # Build for simulator
pnpm ios:run        # Build and run

# Android
pnpm android:assemble  # Build APK
pnpm android:install   # Install on device
pnpm android:run       # Install and launch

# macOS
pnpm mac:package    # Package .app
pnpm mac:restart    # Restart gateway via app
scripts/clawlog.sh  # View unified logs (requires sudo)
```

## Code Structure

### Entry Points
- `openclaw.mjs` - CLI entry wrapper
- `src/entry.ts` - Main entry with process setup and respawning
- `src/cli/run-main.ts` - CLI command dispatcher (Commander.js)
- `src/commands/` - Individual CLI command implementations

### Key Directories
- `src/agents/` - Pi agent core, model integration, tools
- `src/gateway/` - WebSocket server, protocol, connection management
- `src/channels/` - Shared channel logic
- `src/telegram/`, `src/discord/`, etc. - Channel-specific implementations
- `src/config/` - Configuration system
- `src/infra/` - Infrastructure utilities (env, logging, process management)
- `src/media/` - Media pipeline (images, audio, video)
- `src/memory/` - Memory/context management
- `src/sessions/` - Session storage and management
- `src/terminal/` - CLI UI (progress bars, tables, colors)
- `src/utils/` - Shared utilities
- `docs/` - Documentation (Mintlify)
- `extensions/` - Plugin workspace packages

### TypeScript Configuration
- Target: ES2022, module: NodeNext
- Strict mode enabled
- Output: `dist/`
- Tests excluded from compilation

## Coding Guidelines

### Code Style
- TypeScript (ESM only), strict typing, avoid `any`
- Keep files under ~500-700 LOC (guideline, not hard limit)
- Add brief comments for tricky/non-obvious logic
- Use existing patterns: dependency injection via `createDefaultDeps`
- CLI progress: use `src/cli/progress.ts`, not hand-rolled spinners
- CLI colors: use shared palette in `src/terminal/palette.ts`
- Status output: use `src/terminal/table.ts` for tables

### SwiftUI (iOS/macOS)
- Prefer `Observation` framework (`@Observable`, `@Bindable`)
- Don't introduce new `ObservableObject` unless required for compatibility
- Migrate existing `ObservableObject` when touching related code

### Tool Schema Guardrails
- Avoid `Type.Union`, `anyOf`, `oneOf`, `allOf` in tool schemas
- Use `stringEnum`/`optionalStringEnum` for string lists
- Use `Type.Optional(...)` instead of `... | null`
- Keep top-level schema as `type: "object"` with `properties`
- Avoid raw `format` property names (treated as reserved keyword)

### Naming Conventions
- Product/docs: **OpenClaw**
- CLI command/package/binary/paths/config keys: `openclaw`

### Dependencies
- Never update Carbon dependency
- Patched dependencies must use exact version (no `^`/`~`)
- Dependency patches require explicit approval
- Runtime baseline: Node 22+
- Bun supported for dev/scripts, but Node required for production

## Git Workflow

### Commits
- Use `scripts/committer "<msg>" <file...>` (not manual `git add`/`git commit`)
- Concise, action-oriented messages (e.g., "CLI: add verbose flag to send")
- Group related changes, avoid bundling unrelated refactors
- Changelog: keep latest version at top (no "Unreleased")

### Testing Before Commit
Run full gate before pushing:
```bash
pnpm lint && pnpm build && pnpm test
```

### Multi-Agent Safety
- Do NOT create/apply/drop `git stash` entries unless explicitly requested
- Do NOT create/remove/modify `git worktree` checkouts
- Do NOT switch branches unless explicitly requested
- When multiple agents work concurrently, focus on your changes only
- Auto-resolve formatting-only diffs; ask only for semantic changes

## Configuration & Data

### File Locations
- Web provider credentials: `~/.clawdbot/credentials/`
- Pi sessions: `~/.clawdbot/sessions/`
- Agent sessions: `~/.clawdbot/agents/<agentId>/sessions/*.jsonl`
- Environment variables: `~/.profile`

### Security
- Never commit real phone numbers, videos, or live config values
- Use obviously fake placeholders in docs/tests/examples
- Web provider re-auth: `openclaw login`
- Diagnostic tool: `openclaw doctor`

## Platform-Specific Notes

### macOS
- Gateway runs ONLY as the menubar app (no separate LaunchAgent)
- Restart via app or `scripts/restart-mac.sh`
- Verify/kill: `launchctl print gui/$UID | grep openclaw` (not fixed label)
- Logs: `./scripts/clawlog.sh` (queries unified logs)
- No SSH rebuilds; must run directly on Mac
- iOS Team ID lookup: `security find-identity -p codesigning -v`

### Windows
- WSL2 strongly recommended
- Use proper quoting for paths with spaces in Bash commands

### Canvas/A2UI
- Bundle hash: `src/canvas-host/a2ui/.bundle.hash` (auto-generated)
- Only regenerate via `pnpm canvas:a2ui:bundle` or `scripts/bundle-a2ui.sh`
- Commit hash separately when needed

## Release & Publishing

### Release Channels
- **stable:** Tagged releases (`vYYYY.M.D`), npm dist-tag `latest`
- **beta:** Prerelease tags (`vYYYY.M.D-beta.N`), npm dist-tag `beta`
- **dev:** Moving head on `main`

### Version Locations
- `package.json`
- `apps/android/app/build.gradle.kts` (versionName/versionCode)
- iOS/macOS Info.plist files (CFBundleShortVersionString/CFBundleVersion)
- `docs/install/updating.md` (pinned npm version)
- `docs/platforms/mac/release.md` (examples)

### Release Process
- Always read `docs/reference/RELEASING.md` and `docs/platforms/mac/release.md` first
- Do NOT change version numbers without explicit consent
- Always ask permission before npm publish/release steps
- After merging PR from new contributor: run `bun scripts/update-clawtributors.ts`

## Documentation

### Mintlify Docs
- Hosted at https://docs.openclaw.ai
- Internal links: root-relative, no `.md`/`.mdx` (e.g., `[Config](/configuration)`)
- Anchors: root-relative paths (e.g., `[Hooks](/configuration#hooks)`)
- Avoid em dashes and apostrophes in headings (breaks Mintlify anchors)
- README uses absolute URLs (for GitHub)
- Keep content generic: no personal device names/paths

## Common Patterns

### Dependency Injection
Use `createDefaultDeps()` pattern for CLI options and service wiring.

### Error Handling
- Avoid over-engineering; only validate at system boundaries
- Don't add error handling for scenarios that can't happen
- Trust internal code and framework guarantees

### Avoiding Over-Engineering
- Don't add features beyond what's requested
- Don't create abstractions for one-time operations
- Don't add comments/docstrings to unchanged code
- Don't add backwards-compatibility hacks for removed code
- Delete unused code completely

### Messaging Channels
When adding/modifying connection providers:
- Update ALL UI surfaces (macOS app, web UI, mobile if applicable)
- Update onboarding/overview docs
- Add status + configuration forms
- Keep provider lists in sync everywhere

## Known Gotchas

- Voice wake forwarding: `VoiceWakeForwarder` already shell-escapes `${text}`
- Manual `openclaw message send` with `!`: use heredoc pattern to avoid Bash escaping
- Never send streaming/partial replies to external messaging (WhatsApp, Telegram)
- Agent "session" file = Pi session logs, NOT `sessions.json`
- Bug investigations: read npm dependency source code before concluding
