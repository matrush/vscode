# CLAUDE.md

> Code - OSS: the open-source codebase behind Visual Studio Code — a TypeScript code editor for desktop (Electron), web, and remote/server environments.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is the **Code - OSS** repository (the open-source base of Visual Studio Code), built with TypeScript on a layered architecture spanning web, Electron desktop, and remote/server environments. A sibling AI-focused guide with the full coding guidelines lives at [.github/copilot-instructions.md](.github/copilot-instructions.md) — read it for the complete style/quality rules; this file summarizes the highest-leverage parts and the commands.

## Environment

- Node version is pinned in [.nvmrc](.nvmrc) (currently `24.15.0`). Dependencies build against Electron headers (see [.npmrc](.npmrc)), so `npm install` runs native builds.

## Build & Type-Checking

Do **not** type-check changes with `npm run compile`. Instead:

- **Preferred (incremental watch):** run `npm run watch` (or `npm run watchd` to run it as a background daemon, `npm run kill-watchd` to stop). This transpiles core, client, and extensions incrementally; check its output for errors.
- **One-shot type-check of `src/`:** `npm run compile-check-ts-native` — validates `./src/tsconfig.json` (no emit). Use this in CLI/headless environments.
- **Built-in extensions (`extensions/`):** `npm run gulp compile-extensions`.
- **`build/` folder TypeScript:** `cd build && npm run typecheck`.

**Always resolve all compilation errors before running tests or declaring work done. Never run tests while compilation is failing.**

## Linting & Validation

- `npm run eslint` — lint TypeScript (config in [eslint.config.js](eslint.config.js), plus the local plugin in `.eslint-plugin-local/`).
- `npm run stylelint` — lint CSS.
- `npm run hygiene` — copyright headers, indentation (tabs), and formatting checks (also runs as the `precommit` hook).
- `npm run valid-layers-check` — enforce architectural layering (browser/worker/node/electron boundaries). Run this when touching imports across layers.
- `npm run monaco-compile-check` / `npm run vscode-dts-compile-check` — verify the public Monaco editor and `vscode.d.ts` API surfaces still compile.

## Testing

Run test scripts from the `scripts/` folder (the root `npm test` only prints a reminder):

- **Unit (Node):** `scripts/test.sh` (`scripts\test.bat` on Windows). Filter with `--grep <pattern>` to run a single test/suite.
- **Unit (browser):** `npm run test-browser` (installs Playwright first) or `npm run test-browser-no-install`.
- **Integration:** `scripts/test-integration.sh`. Integration tests are files ending in `.integrationTest.ts` or anything under `extensions/`. Variants exist for remote (`test-remote-integration.sh`) and web (`test-web-integration.sh`).
- **Extension tests (`vscode-test`):** `npm run test-extension` (config in [.vscode-test.js](.vscode-test.js)).
- **Smoke tests:** `npm run smoketest`.

Unit tests live next to sources in `src/vs/*/test/` folders.

## Running the Editor

After a build, launch with the scripts in `scripts/`:

- `./scripts/code.sh` — desktop (Electron) build.
- `./scripts/code-web.sh` — VS Code for the Web in a browser.
- `./scripts/code-server.sh` — server / remote setup.
- `./scripts/code-sessions-web.sh` — the Agents Window (see `vs/sessions` below).

## Architecture

The code in `src/vs/` is organized as **strict layers**, each only importing from layers below it. ESLint enforces these boundaries (`valid-layers-check`):

- **`base/`** — foundation utilities and cross-platform abstractions (collections, async, lifecycle, DOM helpers).
- **`platform/`** — platform services and the **dependency-injection** infrastructure. Services are defined here and injected via constructor parameters.
- **`editor/`** — the Monaco text editor: language services, syntax highlighting, editing primitives. Has a public API surface (`monaco.d.ts`).
- **`workbench/`** — the main application shell for web and desktop:
  - `workbench/browser/` — core UI: parts, layout, actions.
  - `workbench/services/` — service implementations.
  - `workbench/contrib/` — feature contributions (git, debug, search, terminal, chat, …). Most feature work happens here.
  - `workbench/api/` — the extension host and the implementation of the `vscode` extension API.
- **`code/`** — Electron main-process code.
- **`server/`** — server-specific code.
- **`sessions/`** — the **Agents Window** (fork-specific). A dedicated, fixed-layout workbench optimized for agent session workflows. It sits alongside `workbench/` and **may import from `workbench/` (and below), but `workbench/` must never import from `sessions/`.** See [src/vs/sessions/README.md](src/vs/sessions/README.md) and its `LAYERS.md`/`LAYOUT.md`/`SESSIONS.md` companions before working there.

Cross-cutting principles:

- **Dependency injection** — declare service dependencies as constructor parameters (decorated). Non-service parameters must come *before* service parameters. Never reach for a service via `IInstantiationService` outside the constructor.
- **Contribution model** — features register into registries / extension points rather than being wired in directly.
- A file's allowed environment (`browser`, `node`, `electron-*`, `worker`) is part of the layering contract — keep platform-specific code behind abstractions.

### Built-in Extensions (`extensions/`)

First-party extensions that ship with VS Code (language features like `typescript-language-features/`, `git/`, `emmet/`, themes, `vscode-api-tests/`, etc.). Each is a standard VS Code extension with its own `package.json` and contribution points. The bundled `extensions/copilot` extension has its own compile/watch lifecycle (`compile-copilot` / `watch-copilot`).

### Other top-level folders

- `build/` — gulp tasks, CI/CD tooling, layering and hygiene checkers. Has its own `package.json`.
- `cli/` — the Rust CLI.
- `test/` — integration/smoke test harnesses and infrastructure.
- `scripts/` — dev launch and test scripts.
- `out/` — generated compiled output (do not edit).

## Key Conventions

Full rules are in [.github/copilot-instructions.md](.github/copilot-instructions.md). The ones most likely to trip you up:

- **Tabs, not spaces.** Every file needs the Microsoft copyright header.
- **Naming:** PascalCase for types and enum values; camelCase for functions, methods, properties, locals.
- **Visibility:** don't `export` types/functions unless shared across components; don't add to the global namespace.
- **Localization:** user-facing strings must be externalized via `vs/nls` (e.g. `nls.localize()`), using `{0}` placeholders rather than concatenation. Use "double quotes" for externalized strings, 'single quotes' otherwise. UI command/button/menu labels use title-case capitalization.
- **Style:** prefer `async`/`await` over `.then()`; prefer top-level `export function x() {}` over `export const x = () => {}` (better stack traces); always brace loop/conditional bodies; prefer arrow functions over `bind/call/apply` for `this` capture.
- **Disposables:** register every disposable for disposal immediately (`DisposableStore`, `MutableDisposable`, `DisposableMap`). Do not register to the containing class if created inside a repeatedly-called method — return an `IDisposable` and let the caller own it.
- **Don't** use `any`/`unknown` unless unavoidable; reuse existing imports (no duplicates) and existing utilities instead of reimplementing.
- **Editors/watchers/tooltips:** open editors via `IEditorService` (not `IEditorGroupsService.activeGroup.openEditor`); prefer correlated file watchers (`fileService.createWatcher`); use `IHoverService` for tooltips.
- **Don't** reach into another component's storage keys to mutate it — add proper API instead.
- **Tests:** prefer a single snapshot-style `assert.deepStrictEqual` over many fine-grained assertions; add tests inside the relevant existing suite; make dependencies injectable rather than stubbing globals or using `any` casts.
- Clean up any temporary scratch files you create before finishing.
