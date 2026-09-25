# ADA Nova Plus / SisAIP — Project Context

## Project Overview

ADA Nova Plus (SisAIP) is an integrated platform for Infoplaza management in Panama,
designed for offline-first local operation and optional central cloud synchronization.

## Components

1. **Soft_Dinamizador**: Desktop application for Dinamizador/admin PC.
   - Path: `Soft_Dinamizador`
   - Repository: independent Git repository.
   - Current local integration branch: `master`, exactly at `7e320970b29286afdc61fc4409764ea90574930f`; PR-02 is integrated there. This is local repository metadata, not a claim about an external remote default.
   - Stack: Electron 33, React 18, Vite 6, TypeScript 5.7, Zustand 4.5, better-sqlite3.
   - Architecture: workspace-style project (`apps/desktop`, `packages/shared-types`). Renderer communicates strictly via IPC bridge (`preload.ts`) to `ipcMain` handlers and SQLite repositories.
2. **Soft_Usuario_PC**: Client agent installed on public user stations.
   - Path: `Soft_Usuario_PC/agente-infoplaza`
   - Repository: independent Git repository.
   - Default integration branch: `master` until explicitly changed; there is no ADA Nova Plus common `main`.
   - Stack: Go 1.22, Wails v2 (2.12.0), React 18, Vitest frontend test runner.
   - Function: Station lock/unlock, user session control, time tracking, offline logging.
3. **Página Web Nacional**: Future central web and reporting platform (out of scope for current local implementation phase).

## Normative Hierarchy

1. `docs/DINAMIZADOR - REQUERIMIENTOS PARA DESARROLLO.pdf` (Governs Soft_Dinamizador)
2. `docs/USUARIO PC - REQUERIMIENTOS PARA DESARROLLO.pdf` (Governs Soft_Usuario_PC)
3. `docs/REQUERIMIENTOS PARA DESARROLLO.docx` (Platform & web context)
4. `docs/GUARDRAILS_DE_INTEGRACION_ADA_NOVA_PLUS.md` (Integration guardrails)

## Architectural Guardrails

- **Repository model**: ADA Nova Plus currently uses independent component repositories. `stacked-to-main` means stacked to each repository's own local integration branch (`master` for Usuario PC and Dinamizador today), not to one shared ADA Nova Plus mainline. Cross-component prerequisites are OpenSpec cross-repository contract dependencies, never Git ancestry across repositories.
- **Operational authority**: `C:/SisAIP` is the planning/orchestration root for OpenSpec, requirements, architecture, cross-repository decisions, and PR planning. Implementation, tests, `sdd-apply`, `git status`/`git diff`, commits, and PR review must run from the affected PR's Git worktree. Before implementation or commit, confirm the correct repo, branch, worktree, baseline, and expected working tree/diff. If started from `C:/SisAIP` and a Git operation needs native repo authority, stop and resume from the proper worktree; do not bypass.
- **Code Evolution**: Standard decision order: `KEEP > EXTEND > REFACTOR > REPLACE > ADD`. Rewrites are prohibited by default.
- **Stable Identifiers**: Primary technical IDs must be internal UUIDs/IDs. Document numbers (cédula/pasaporte), MAC addresses, and PC names are search/display criteria only.
- **Offline-First**: Stations and Dinamizador operate locally over the LAN without requiring persistent Internet. mDNS may support discovery, but trusted privileged Dinamizador ↔ Usuario PC traffic MUST use WSS over TLS 1.3 with mutual TLS and locally trusted, unexpired credentials; plaintext WS is not an authorized path.
- **Open Requirements**: Any undefined functional requirement must be explicitly marked as `OPEN REQUIREMENT`.
- **Compatibility boundary**: Legacy parsing MAY remain for transition, but parser compatibility MUST NOT authorize privileged mutations, permit a v2-failure-to-legacy downgrade, or bypass the authenticated secure-link contract.
- **Auditing**: All key session and operational state changes must emit traceable audit events.

## Testing Setup

- **Soft_Usuario_PC Frontend**: Vitest runner in `Soft_Usuario_PC/agente-infoplaza/frontend` (`npm test`).
- **Soft_Usuario_PC Backend**: Go test runner (`go test ./...`).
- **Soft_Dinamizador**: PR-02 main-process test harness is integrated on the current local `master`; future modules remain subject to strict TDD.
