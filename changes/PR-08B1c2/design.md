# PR-08B1c2 ACTIVE / Lifecycle Umbrella

## Status: POLICY DIRECTION APPROVED — IMPLEMENTATION BLOCKED

## Decision

B1c2 is the eventual activation-capable boundary after B1c1a1, B1c1a2, and
B1c1b integrate. It is an umbrella, not a runtime candidate. Its descendants
may be proposed only after the mandatory Clock Plausibility Decision and fresh
integrated-code inspection:

```text
B1b2b2 COMPLETE + INTEGRATED
  -> B1c1a1 COMPLETE + INTEGRATED
  -> B1c1a2 COMPLETE + INTEGRATED
  -> B1c1b COMPLETE + INTEGRATED
  -> Clock Plausibility Decision
  -> B1c2 lifecycle umbrella
  -> future security/lifecycle-cohesive descendants
```

See [B1c](../PR-08B1c/design.md) and [B1c1b](../PR-08B1c1b/design.md).
The activation-capable B1c2 boundary remains unimplemented and unauthorized.

## Integrated non-activating Clock Evidence admission

Dinamizador PR #22, `feat(security): add clock evidence admission`, was merged
at `e4f35a086a9794e43d1d49619380d1134e70fac2`. It integrates a bounded,
non-activating Clock Evidence admission refinement. Its Quality CI completed
successfully, and its reported repository verification was 378/378 passing
checks.

The merged scope records only these Clock Evidence behaviors:

- private persistence advances the accepted high-water wall-clock value
  monotonically;
- rollback detection uses a 300,000 ms tolerance;
- store advancement uses interprocess exclusion and fails closed when the lock,
  storage, or protected storage cannot establish the required result; and
- interrupted-write recovery is restricted to quiescent cleanup of inherited
  lock artifacts. It does not infer ownership, restore authority, or perform
  automatic recovery.

This evidence does **not** authorize `link.accept`, `ACTIVE`, institutional
`connection_epoch` generation, operational admission, a 30-day policy,
student-session activation, or PR-08U1. It does not settle institutional time
authority, offline/restart validity, provenance, replay, recovery, certificate
rotation, or revocation. No independent integration evidence is recorded here
for B1c2a3 / `CLOCK_EVIDENCE` audit delivery; that work remains pending.

## Blocking clock decision

Clock Plausibility remains OPEN. It does not block B1c1a1, B1c1a2, or B1c1b's
internal non-active authorization/negotiation. It does block final
activation-capable accept and ACTIVE/production acceptance. A locally
implausible/untrustworthy clock must prevent a new ACTIVE transition and cannot
bypass X.509 `notBefore`/`notAfter` validation because the endpoint is offline
([`spec.md:383-409`](../session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md)).

Before B1c2 authorization, the decision must settle local-clock evidence,
certificate-validity/expiry interaction, offline rollback limitation,
failure/close/reconnect behavior, local event/audit ownership without promising
durable B2 persistence, and the activation boundary that can emit accept.

Only then may a new inspection assess the integrated B1c1a1 codec, B1c1a2
handoff, B1c1b composition, authorization flow, and current FSM/epoch/sequence
facilities for cohesive lifecycle seams. `sourceId` semantics remain OPEN and
are not reopened by B1c2 planning.

## Future activation requirements

A later B1c2 child, never B1c1a1/a2/b, must:

- accept the B1c1b authorized/negotiated non-active result only after clock
  plausibility and current credential validity pass;
- generate a cryptographically random, nonzero, fresh `connection_epoch` for
  every successful connection/reconnect, never precompute/reserve it earlier;
- preserve exact uint64 transport semantics without lossy JavaScript `Number`.
  Existing bigint validation does not settle JSON serialization
  (`connection-epoch.ts:3-27`);
- construct and emit the exact correlated final `link.accept` only at this
  boundary; only verified accept may lead Usuario PC to ACTIVE; and
- apply post-ACTIVE directional sequence/replay guards only after handshake,
  which has no sequence.

Normative sources are [`spec.md:255-317`](../session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md)
(hello/accept) and [`spec.md:351-445`](../session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md)
(epoch, sequence, clock, lifecycle, zero product). The current FSM lacks this
complete proof (`fsm.ts:8-79`); neither it nor a legacy refactor is an
activation substitute or authorized B1c2 work.

Future children must keep activation prerequisites; active sequence/replay/gap/
overflow, replacement/stale close/cleanup, reconnect reset, and expiry; and
local event/audit plus later durable-audit seam security-cohesive. Product
behavior remains forbidden before ACTIVE. `link.reject` is still post-mTLS,
safe-correlation-only, and epoch/sequence/ACTIVE-free; TLS failures are local,
and installation/credential deny remains close-only without a new wire code.

## Contingent forecast and test ownership

Planning estimate: new production `+145/-0`, existing `+35/-8`, tests
`+240/-0`, total `+420/-8`, **NET 412**, **CHURN 428**. It is not measured and
**NOT VIABLE** as one <=300-NET candidate; the future <=400 actual gate cannot
rescue it. After the clock decision and full B1c1 integration, a proposal must
supply new paths, ownership, forecast, and tests before writing code.

Later tests must cover activation/clock/credential prerequisites; fresh epoch
per reconnect and uint64 preservation; sequence/replay/gap/overflow,
replacement/stale close/cleanup/expiry; audit-event versus durable-audit
boundary; and no business behavior before ACTIVE. No runtime test is created
by this documentation-only umbrella.
