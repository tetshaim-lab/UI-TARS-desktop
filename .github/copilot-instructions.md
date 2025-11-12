## Quick guide for AI coding agents working on UI-TARS-desktop

This file contains concise, discoverable facts to help an AI agent be immediately productive in this repository.

> **Security note:** Do NOT include secrets (API keys, tokens, private URLs, .env contents) in this file or PR descriptions.
> **Before opening a PR:** run `pnpm i`, `pnpm typecheck`, `pnpm lint`, `pnpm test`.
> See `CONTRIBUTING.md` for contributor guidelines.

- Project type: a pnpm monorepo (Node >= 20) containing an Electron + Vite desktop app in `apps/ui-tars` and many workspace packages under `packages/`.
- Key app: `apps/ui-tars` (package name: `ui-tars-desktop`). Major entry points:
  - Main process: `apps/ui-tars/src/main/main.ts` (app bootstrap, IPC registration)
  - Preload context: `apps/ui-tars/src/preload/index.ts` (contextBridge exposes `electron`, `zustandBridge`, `platform`)
  - Renderer root: `apps/ui-tars/src/renderer/src/main.tsx` and `apps/ui-tars/src/renderer/src/*` (React + Vite + Tailwind)

Architecture notes (big picture)
- Monorepo layout: root `package.json` uses pnpm/turbo; packages use `workspace:*` deps. Shared code lives under `packages/` (e.g. `@ui-tars/electron-ipc`, `@ui-tars/sdk`, `agent-infra`).
- IPC and state: The app uses an explicit preload -> contextBridge pattern. Use `window.electron` and `window.zustandBridge` in renderer code. IPC routes are implemented in `apps/ui-tars/src/main/ipcRoutes/*.ts` and registered via `registerIpcMain(ipcRoutes)` in `main.ts`.
- State sync: The app uses a zustand bridge implemented in preload: `zustandBridge.getState()` and `zustandBridge.subscribe(...)` are used to sync main/renderer state.
- External integration: MCP servers and operators live in `packages/agent-infra*` and `packages/ui-tars/operators/*` (see `packages/agent-infra`, `packages/ui-tars/operators/*`). Use `packages/agent-infra/create-new-mcp/template-default/README.md` as a template for creating new MCP servers.

Common developer workflows & exact commands
- Requirements: Node >= 20, pnpm (repo uses `pnpm@9.10.0` in `package.json`).
- Install: from repo root run:
  ```bash
  pnpm i
  ```
- Run the desktop app during development (two options):
  - From monorepo root (shortcut script):
    ```bash
    pnpm run dev:ui-tars
    ```
  - Or from package dir:
    ```bash
    cd apps/ui-tars
    pnpm dev        # runs `electron-vite dev`
    pnpm debug      # runs `electron-vite dev --sourcemap --remote-debugging-port=9222`
    ```
- Build & package (from `apps/ui-tars`):
  ```bash
  cd apps/ui-tars
  pnpm run build       # full build (typecheck + electron-vite build + electron-forge make)
  pnpm run make        # electron-forge make
  ```
- Tests: unit tests use Vitest; e2e uses Playwright.
  - Run unit tests (root or package): `pnpm test` or `cd apps/ui-tars && pnpm test`
  - Run e2e: `cd apps/ui-tars && pnpm run test:e2e` (script uses `playwright test`)
- Lint/format: `pnpm run lint` and `pnpm run format` (available in root and packages)

Patterns & conventions to follow (code-level)
- Split responsibilities: change main-process code under `apps/ui-tars/src/main/*`; UI changes go under `apps/ui-tars/src/renderer/src/*`; use `preload/index.ts` to expose safe APIs.
- When adding IPC handlers: add a route in `apps/ui-tars/src/main/ipcRoutes/*` and ensure it is exported in the `ipcRoutes` collection used by `registerIpcMain` in `main.ts`.
- Use workspace packages for cross-cutting functionality. Example: shared IPC helpers live in `packages/ui-tars/electron-ipc` (see `registerIpcMain` import in `main.ts`).
- Typechecking is split: `tsconfig.node.json` for node/main and `tsconfig.web.json` for renderer. Use provided npm scripts `typecheck:node` and `typecheck:web` when changing types.

Files to consult for examples
- App bootstrap and IPC: `apps/ui-tars/src/main/main.ts`
- Preload / contextBridge: `apps/ui-tars/src/preload/index.ts`
- Renderer entry and React layout: `apps/ui-tars/src/renderer/src/main.tsx`, `apps/ui-tars/src/renderer/src/App.tsx`
- Shared IPC module: `packages/ui-tars/electron-ipc` (see how main and renderer share handlers)
- Electron build and config: `apps/ui-tars/electron.vite.config.ts`, `apps/ui-tars/forge.config.ts`, root `electron.vite.config.ts`
- Docs and quick start: `docs/quick-start.md`, `docs/sdk.md`, root `README.md`

Practical examples for patches
- To expose a new action to renderer:
  1. Add handler file `apps/ui-tars/src/main/ipcRoutes/myAction.ts`.
  2. Export it from `apps/ui-tars/src/main/ipcRoutes/index.ts` (so `ipcRoutes` picks it up).
  3. Call it from renderer via the preload-exposed API: `window.electron.ipcRenderer.invoke('myAction', params)` or implement a safe wrapper on `preload/index.ts` if needed.

Notes & gotchas
- Many internal packages reference `workspace:*` — run `pnpm i` at the root to ensure symlinked packages are resolved.
- CI and publishing use scripts under `scripts/` and `turbo` from the root; avoid ad-hoc package version changes without updating release scripts.
- Keep changes small and run `pnpm run typecheck` and `pnpm run lint` before opening PRs.

If anything in this file looks incorrect or incomplete, tell me which area to expand (architecture, scripts, IPC, tests) and I will iterate.
