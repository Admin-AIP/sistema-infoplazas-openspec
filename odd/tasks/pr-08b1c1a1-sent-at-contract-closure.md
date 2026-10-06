# PR-08B1c1a1 sent_at contract closure

## Authorized scope

OpenSpec contract reconciliation only. Do not modify Dinamizador or implement B1c1a1, B1c1a2, or B1c1b.

## Verified base

- Dinamizador `origin/master`: `bcef13adde52b59b462c5096824be73b3d5295d9`
- Dinamizador tree: `89fb124de893dfc41e1e7511b1aeb1ea560bbdd9`
- OpenSpec `origin/master`: `5ab590d463d64269f62dba42b8dea1bc9493cabe`

## Task

- [x] Define the exact ADA PR-07 `sent_at` wire profile in the minimum authoritative OpenSpec surfaces, preserve all dependency and security boundaries, independently review consistency, and deliver one documentation commit to `origin/master`.

## Required verification

- PR-07 generic design, secure-link specification, and B1c1a1 design state the same exact timestamp profile.
- `correlation_id` remains optional for requests and exact-preserved when present.
- Unknown outer fields remain inertly tolerated; the hello payload remains exact.
- Clock Plausibility and `sourceId` semantics remain OPEN.
- B1c1a1 receives no implementation authorization; B1c1a2 and B1c1b remain blocked.
- The prior 278-NET projection remains provisional and requires a new fresh checkpoint.
- `git diff --check` passes.

## Evidence

- Independent consistency review: PASS via read-only explorer fallback; no blockers.
- Native assessment: unavailable because untracked files required explicit declaration; no native review evidence is claimed.
- Package verifier: unavailable after two failed attempts; recorded truthfully.
- Structural check: `git diff --check` exited 0.
- Commit identity: `docs(openspec): close PR-07 sent_at profile`.
- Delivery target: normal push to `origin/master` without force.
