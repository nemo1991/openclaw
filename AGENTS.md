# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenClaw is a multi-channel AI gateway with extensible messaging integrations — an AI that runs on your devices, in your channels, with your rules. The core stays plugin-agnostic; optional capability ships as plugins.

- **Repo**: `https://github.com/openclaw/openclaw`
- **Runtime**: Node 22+ (also Bun-compatible)
- **CLI**: `pnpm openclaw ...` or `pnpm dev`
- **Version**: `2026.5.14`

## Architecture

### Directory Structure

```
src/
├── acp/           # Agent Communication Protocol — session lifecycle, event ledger, translator
├── agents/         # Agent runtime, spawn, auth profiles, bash tools, transport streams
├── channels/       # Channel-agnostic session/sending/targeting infrastructure
├── gateway/        # Gateway server, auth, config, client bootstrap, control UI
├── plugins/        # Plugin loader, registry, bundling, capability metadata
│
src/plugin-sdk/    # Public SDK for plugin authors (exported as `openclaw/plugin-sdk`)
src/channels/      # Channel implementation under src/channels
extensions/        # Bundled and external plugins (discord, telegram, slack, etc.)
packages/          # Sub-packages (memory-host-sdk, plugin-package-contract, sdk)
ui/                # Web UI components
apps/              # Native apps (macos, ios, android)
docs/              # Mintlify-hosted documentation
```

### Key Concepts

- **ACP (Agent Communication Protocol)**: Session lifecycle, event ledger, translator. See `src/acp/`.
- **Plugin SDK**: `openclaw/plugin-sdk/*` — public surface for plugin authors. Plugins cross into core only via this SDK + manifest metadata + injected runtime helpers.
- **Gateway**: `src/gateway/` — HTTP server handling auth, config, client bootstrap, control UI.
- **Channels**: Implementation under `src/channels/**`; plugin authors get SDK seams.
- **Agents**: `src/agents/` — Agent runtime, spawn system, auth profiles, bash tools.
- **Bundled plugins**: Ship in core dist under `extensions/`. External plugins (discord, telegram, etc.) are excluded from core dist and use registry-aware facade-runtime.
- **ACPX**: Extended ACP with additional capabilities.

### Hot Path Guidance

Hot paths should carry prepared facts forward: provider id, model ref, channel id, target, capability family, attachment class. Do not rediscover with broad plugin/provider/channel/capability loaders.

### Dependency Ownership

- Plugin-only deps stay plugin-local
- Root deps only for core imports or intentionally internalized bundled plugin runtime
- Owner-specific behavior (repair/detection/onboarding/auth) lives in owner plugin; core gets generic seams only

## Common Commands

```bash
# Development
pnpm install          # Install dependencies
pnpm dev             # Run CLI in dev mode
pnpm build           # Build all packages

# Testing
pnpm test <path>           # Run tests (colocated *.test.ts)
pnpm test:changed          # Test only changed files
pnpm test:serial           # Run tests serially (avoids Vitest cache races)
pnpm test:coverage         # With coverage
pnpm test:extensions       # Test all extensions
pnpm test extensions/<id> # Test specific extension
pnpm test:contracts       # Plugin/channel contract tests

# Type checking
pnpm check:changed         # Check changed files
pnpm check                 # Full check

# Formatting / Linting
pnpm format                # oxfmt write
pnpm lint                  # Run linters
pnpm lint:core             # Lint core only

# Extension tests
pnpm test:extension <name> # Test specific extension
pnpm test:extension --list # List valid extension ids
```

## Code Conventions

- **TS ESM, strict mode**. Avoid `any`; prefer real types, `unknown`, narrow adapters.
- **Runtime branching**: discriminated unions/closed codes over freeform strings.
- **Dynamic imports**: use `*.runtime.ts` lazy boundary for same module.
- **Classes**: no prototype mixins/mutations. Prefer inheritance/composition.
- **Comments**: brief, only non-obvious logic.
- **Naming**: **OpenClaw** product/docs; `openclaw` CLI/package/path/config.
- **Split files** around ~700 LOC when clarity/testability improves.

## Plugin Development

- Plugins cross into core via `openclaw/plugin-sdk/*`, manifest metadata, injected runtime helpers.
- Plugin prod code: no core `src/**`, other plugin `src/**`, or relative outside package.
- External boundaries: prefer `zod` or existing schema helpers.
- Provider tool schemas: prefer flat string enum helpers over `Type.Union([Type.Literal(...)])`.

## Gateway Protocol

Gateway protocol changes: additive first; incompatible needs versioning/docs/client follow-through.

## Secrets

- Channel/provider creds: `~/.openclaw/credentials/`
- Model auth profiles: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Never commit real credentials, phone numbers, or live config.

## Additional Context

- **AGENTS.md** exists at root and in scoped directories (e.g., `src/channels/AGENTS.md`, `src/plugins/AGENTS.md`). Read scoped AGENTS.md before subtree work.
- **Control UI** uses Lit with legacy decorators (`@state()`, `@property()`).
- **Mac gateway dev**: `pnpm gateway:watch`; managed installs: `openclaw gateway restart/status --deep`
- **SwiftUI**: Use `@Observable`, `@Bindable` over `ObservableObject`.
