# ADR: Institutional signed-time provenance and relay direction

## Status: POLICY DIRECTION SELECTED — institutional approvals pending

## Decision

The functional policy direction is an institutional time authority that signs
time/provisioning evidence. The Dinamizador MAY cache and relay that signed
evidence to local consumers, but it is never a time authority and MUST NOT
mint, extend, repair, or reinterpret signed time.

TPM continuity is excluded from the baseline. It may be assessed later as an
optional defense-in-depth signal for inventoried, supported hardware; it is
not a UTC authority and cannot replace institutional signed evidence.

This ADR records a direction only. It authorizes no runtime code, storage,
issuer, distribution service, recovery, audit sink, `link.accept`, epoch,
`ACTIVE`, User PC behavior, product action, or wire change.

## Security invariants

- Independent X.509 `notBefore`/`notAfter` validation remains mandatory and
  cannot be bypassed by a plausibility decision or relay response.
- Only evidence verifiable against an approved institutional trust anchor may
  influence a future activation-capable clock decision.
- Dinamizador relay/cache failure, unavailable evidence, failed verification,
  uncertain continuity, or expired evidence MUST fail closed for a *new*
  activation-capable transition. They do not authorize a repair or a trusted
  time claim.
- Signed evidence, if later specified, must be bound to its intended
  installation and certificate identity and protected against replay.
- Existing non-active B1c1a1, B1c1a2, and B1c1b behavior remains unchanged.

## Decisions intentionally deferred

The following are not selected by this ADR:

1. signing format, trust-anchor delivery, key custody, issuer/service owner,
   certificate rotation, revocation, or compromise response;
2. evidence lifetime, offline window, drift/rollback tolerance, clock sources,
   freshness, or check timing;
3. behavior for already-`ACTIVE` links, reconnect, process restart, CMOS
   battery failure, recovery, or endpoint symmetry;
4. persistence adapter, relay cache retention, audit vocabulary, audit
   retention, access, export, deletion, collision handling, alerting, or
   operational ownership; and
5. TPM enrollment, attestation, inventory, firmware, replacement, or fallback.

No unapproved candidate value, including the historic five-minute rollback
candidate or a 30-day snapshot candidate, is policy authority.

## Required consultations

| Owner | Required decision before implementation |
| --- | --- |
| Product / Architecture | activation boundary, check timing, existing-`ACTIVE` behavior, and acceptance criteria |
| Security | evidence format, signer/trust-anchor verification, replay, recovery, rotation, revocation, and incident response |
| Institutional time authority and IT Operations | issuer/service ownership, key custody, distribution, offline/restart validity, support, and budget |
| Privacy | audit minimization, retention, access, export, deletion, and compliance basis |
| Usuario PC owner | endpoint symmetry, restart semantics, and any User PC-visible behavior before it is concluded or changed |

Guillermo, the Usuario PC owner, was notified of this direction before any User
PC-affecting conclusion. Notification is coordination, not approval.

## Preconditions for a future implementation proposal

A proposal may be considered only after all required consultations produce
explicitly recorded decisions, the resulting normative contract defines the
remaining deferred items needed by its scope, and a fresh integrated-code
inspection identifies a bounded candidate. Until then, B1c2 remains
implementation-blocked.

## Consequences

This direction provides an institutional provenance path without treating the
Dinamizador as trusted time or lowering the existing fail-closed boundary. It
does not unblock `link.accept`, new `ACTIVE`, production acceptance, or
PR-08U1 implementation.