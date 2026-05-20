# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Vimium C is a cross-browser keyboard-driven extension (Chrome/Edge/Firefox) that allows full page navigation and browser control via keyboard shortcuts. It is a heavily customized fork of [Vimium](https://github.com/philc/vimium).

- Targets Chrome/Chromium 102+, Edge 102+, Firefox 101+
- Supports both Manifest V2 and Manifest V3

## Build System

The project uses **Gulp + TypeScript**. Source `.ts` files compile to `.js` in-place (same directory).

```bash
# Install dependencies
npm install

# Watch mode (incremental TypeScript compilation)
npm run dev
# or
npm run watch

# Full build (all sources + assets)
npm run build
# or
gulp build

# Local dev build (outputs to .build/ directory)
npm run local
# or
gulp local

# Rebuild from scratch
npm run rebuild
# or
gulp rebuild

# Run tests
npm test
# or
gulp test

# Run linter
npm run lint
```

### Distribution builds (produce zip packages via `scripts/make.sh`)

```bash
npm run chrome        # Chrome/Chromium MV3 (latest)
npm run edge          # Edge MV3
npm run mv3-firefox   # Firefox MV3
npm run mv2-cr        # Chrome MV2 (legacy)
npm run mv2-ff        # Firefox MV2
npm run mv2-edge      # Edge MV2
```

### Key build environment variables

| Variable | Default | Meaning |
|---|---|---|
| `BUILD_BTypes` | — | Browser target bitmask: `1`=Chrome, `2`=Firefox, `4`=Edge(old) |
| `BUILD_MV3` | `1` | `0` to build MV2 |
| `BUILD_MinCVer` | `102` | Minimum Chromium version |
| `BUILD_MinFFVer` | `101` | Minimum Firefox version |
| `BUILD_NeedCommit` | — | `1` to require a clean git commit |

## Architecture

The extension is divided into four major script contexts, each compiled independently via TypeScript project references:

### `background/`
Service worker (MV3) / background page (MV2). Handles:
- Settings management (`settings.ts`, `store.ts`)
- Tab and window commands (`tab_commands.ts`, `frame_commands.ts`)
- URL completion and omnibar backend (`completion.ts`, `completion_utils.ts`)
- Key mapping and shortcut dispatch (`key_mappings.ts`, `run_commands.ts`, `run_keys.ts`)
- Port/messaging with content scripts (`ports.ts`, `request_handlers.ts`)
- URL parsing and navigation (`parse_urls.ts`, `normalize_urls.ts`, `open_urls.ts`)

Entry point: `background/main.ts` (imports everything via side-effect imports).

### `content/`
Content scripts injected into every web page. Handles:
- Key event capture and routing (`key_handler.ts`, `frontend.ts`)
- Link hint UI (`link_hints.ts`, `hint_filters.ts`, `link_actions.ts`)
- Scroll (`scroller.ts`), visual mode (`visual.ts`), find mode (`mode_find.ts`)
- Insert mode / input focus (`insert.ts`)
- HUD display (`hud.ts`)
- DOM manipulation helpers (`dom_ui.ts`, `extend_click.ts`)
- Communication with background (`port.ts`, `request_handlers.ts`)

Entry point: `content/frontend.ts` (loaded last per manifest).

### `front/`
Isolated front-end pages running in extension context but displayed as overlay UI:
- `vomnibar.ts` — the omnibar/address bar overlay
- `tee.ts` — message relay for cross-frame communication

### `pages/`
Extension HTML pages:
- `options*.ts` — settings UI (split across multiple files for size)
- `show.ts` — "show page" for history/bookmarks display
- `async_bg.ts` — async proxy to background for options pages

### `lib/`
Shared utilities compiled into content scripts:
- `env.ts` — global environment flags (`OnChrome`, `OnFirefox`, `OnEdge`, `CurCVer_`)
- `utils.ts` — core utilities and shared state
- `keyboard_utils.ts` — key normalization
- `dom_utils.ts` — DOM helpers
- `rect.ts` — geometry utilities
- `polyfill.ts` — browser polyfills for older versions

### `typings/`
TypeScript declaration files only (no runtime output):
- `vimium_c.d.ts` — main message/command type definitions
- `base/base.d.ts` — custom utility types (`BOOL`, `Dict`, `SafeDict`, etc.)
- `base/chrome.d.ts` — browser API typings

## TypeScript Patterns

- **`const enum`** is used extensively for zero-cost integer constants (compiled away entirely). Examples: `kCmdInfo`, `kFgReq`, `kBgReq`, `HandlerResult`.
- **`OnChrome` / `OnFirefox` / `OnEdge`** are compile-time boolean constants from `lib/env.ts`. Code gated on these is dead-eliminated during builds for specific browser targets.
- **Namespace-based modules**: many shared types live in `declare namespace` blocks in `typings/`.
- The build system patches TypeScript's `tsc` binary (`scripts/tsc.js`) to support custom build features like namespace rewriting and dead-code elimination.
- Each directory has its own `tsconfig.json` that extends `tsconfig.base.json`.

## Messaging Architecture

Background ↔ Content communication uses Chrome's `runtime.connect` / `runtime.sendMessage`:
- Content → Background: typed as `kFgReq` enum values
- Background → Content: typed as `kBgReq` enum values
- Frames communicate via a `vApi` object injected into the page world

All message handler registrations and port management live in `background/ports.ts` and `content/port.ts`.
