# PR-08B1c1a delivery split

## Authorized scope

OpenSpec/design revision only. Do not modify Dinamizador or implement B1c1a1, B1c1a2, B1c1b, or B1c2.

## Verified base

- Dinamizador `origin/master`: `bcef13adde52b59b462c5096824be73b3d5295d9`
- Dinamizador tree: `89fb124de893dfc41e1e7511b1aeb1ea560bbdd9`
- OpenSpec `origin/master`: `c057c26f4e54dc30f7b45baa4fcab0102d3934f3`

## Tasks

- [x] Revise the B1c designs to make B1c1a an architectural umbrella delivered by B1c1a1 and B1c1a2, preserve all stated security and readiness boundaries, independently review consistency, and deliver one OpenSpec documentation commit to `origin/master`.

## Required verification

- Dependency links are consistent.
- B1c1b remains not started and not ready.
- Clock Plausibility remains open and blocks final activation-capable acceptance.
- Final accept, epoch generation, and ACTIVE do not move earlier.
- `sourceId` semantics remain open.
- Product actions remain zero.
- Forecast arithmetic is exact.
- `git diff --check` passes.
- Independent consistency review is recorded truthfully.

## Evidence

- Independent consistency review: PASS; no blockers.
- Structural check: `git diff --check` exited 0.
- Native assessment: unavailable because untracked files required an explicit declaration; no native review evidence is claimed.
- Commit identity: `docs(openspec): split B1c1a delivery design`.
- Delivery target: normal push to `origin/master` without force.
