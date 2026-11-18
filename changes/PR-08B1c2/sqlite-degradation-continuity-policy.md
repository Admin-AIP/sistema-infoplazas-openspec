# SQLite degradation continuity policy (supplemental candidate)

## Status and boundary

This is a **review candidate** and supplemental decision record for one isolated
SQLite-continuity question. It does not merge, supersede, or alter the broader
PR-08B1c2 lifecycle policy, clock policy, or any other B1c2 policy.

## Decision

When local SQLite authorization authority is unavailable, corrupt, or
inconsistent, only activity that is already running in a Dinamizador process
may continue, and only when that process previously loaded and verified intact
local authorization.

The degraded condition blocks:

- new authorization-dependent starts or sessions;
- authorization changes;
- role or privilege changes; and
- cloud synchronization.

After a restart, or whenever verified in-memory authorization state is absent,
an authorization-dependent start remains blocked until integrity is recoverable.

Known local deny, X.509 validation, locally known revocation, and every
existing security gate remain mandatory. Degraded continuity is not an
authorization bypass.

## Operator visibility and records

On detecting degradation, plainly notify the operator that local SQLite
authority is degraded, state the affected scope, and state the forbidden
actions. Record detection and recovery where available.

Before clearing or updating the degraded notice, plainly notify the operator of
verified recovery and integrity status.

## Explicit non-decisions

This candidate states no Usuario PC behavior or conclusion. It does not decide
recovery implementation, UI copy or channel, wire protocol, persistent audit
guarantee, runtime code, or commit/publish authorization beyond this isolated
documentation candidate.
