# PR-08B1c2 ACTIVE / Lifecycle Umbrella

## Status: BLOCKED — planning only; NOT IMPLEMENTATION READY

## Decision

B1c2 is the eventual activation-capable boundary after B1c1a1, B1c1a2, and
B1c1b integrate. It is an umbrella, not a runtime candidate. No mechanical
B1c2a/B1c2b split or runtime implementation is authorized today.

See [B1c](../PR-08B1c/design.md) and [B1c1b](../PR-08B1c1b/design.md).
The activation-capable B1c2 boundary remains unimplemented and unauthorized.

## Historical superseded Clock Evidence record

Dinamizador PR #22, `feat(security): add clock evidence admission`, was merged
at `e4f35a086a9794e43d1d49619380d1134e70fac2`; its Quality CI was reported as
378/378 passing. Its non-activating Clock Evidence work recorded monotonic
high-water persistence, a 300,000 ms rollback tolerance, fail-closed store
handling, and limited interrupted-write cleanup. Those are historical facts,
not current policy or authorization: the simplified local policy supersedes the
high-water, rollback, evidence-store, and institutional signed-time direction.

The former limited B1c2a1 authorization request was likewise limited to a pure
local-consistency evaluator with no I/O, store, issuer, distribution, audit,
recovery, FSM, wire, `link.accept`, `connection_epoch`, `ACTIVE`, admission, or
product privilege. It is retained as historical scope only and does not
authorize work under the simplified local policy.

## Simplified local policy

The former institutional signed-time-evidence design is superseded. There is no
institutional time service, signed evidence, refresh cadence, time trust anchor,
offline recovery package, or signed high-water rollback tolerance.

The institutional installer is code-signed only to prove software origin and
integrity. Runtime local authorization relies on valid X.509 credentials using
the OS wall clock, trusted local certificate roots, the local authorization
registry, and locally known deny/revocation/business decisions. Dinamizador's
local SQLite is the sole authorization/deny source during isolation; it stores
only installation/station/center/source identity, role, allow/deny state, credential
fingerprint or serial, version and update marker, and receives online updates
atomically. Usuario PC has no competing authoritative registry. The system does
not claim offline clock-rollback protection. A clock that cannot validate X.509
must prevent `ACTIVE`.

An isolated center may continue local operation and reconnect within its known
center without a time limit while those gates pass. It cannot enroll, replace a
Dinamizador, change center, or synchronize cloud records while isolated.
Replacement/reinstallation requires connectivity for the ordinary center
authorization flow.

Dinamizador uses one local SQLite with separate authorization/deny, queued
record and thirty-day rotating audit tables. Private X.509 keys stay outside
SQLite in Windows CNG/Certificate Store, accessible only to
`NT SERVICE\InfoplazaSecureLink`; SQLite and keys deny shared student accounts.
SQLite changes are atomic. This protects against ordinary users, errors and
accidental copies, not a local Windows administrator.

If the SQLite authorization/deny state becomes unavailable, corrupt, or
inconsistent, Dinamizador-local continuity is bounded: only a process that
already loaded and verified intact local authorization state may retain its
pre-existing local activity. It MUST forbid new sessions or authorizations,
role or privilege changes, and cloud synchronization until recoverable
integrity is restored. After restart, or without that prior verified in-memory
state, authorization-dependent starts MUST be blocked. Known local deny,
X.509 validation, locally known revocation, and every other existing security
gate remain enforced; this degraded state MUST NOT create a bypass. Detection
MUST plainly notify the operator of the degraded state, its continuity scope,
and the forbidden actions; detection and recovery MUST be recorded where
available. Before any degraded notice is cleared or updated, the operator MUST
be plainly notified that recovery and integrity have been verified, including
the recovery status. This policy is Dinamizador-local only and does not conclude any
Usuario PC behavior.

When connectivity returns, Dinamizador authenticates to the cloud and atomically
updates authorization/deny first. Each queued record has an immutable `operationId`
and carries the authorization version under which it was created; the cloud accepts
it once or reports it already received. A record created under an authorized version
is accepted and flagged if authorization was later revoked or its center later
changed. Dinamizador marks delivery only after confirmation, retries
transient/partial failures without duplication, and quarantines conflicts (and local records without a valid creation version) without
deleting or silently uploading them. Quarantined records do not hold later records, except operation types that declare strict ordering. The cloud remains authoritative for
authorization, deny and center changes; quarantined exceptions are reviewed by an
authorized reviewer, with the role to be defined (the Enlace is not the default reviewer).

## Future activation requirements

A later B1c2 child, never B1c1a1/a2/b, must:

- accept the B1c1b authorized/negotiated non-active result only after current
  credential validity, local authorization, local deny/revocation/business gates
  and any applicable local audit gate pass;
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
rescue it. After the historical superseded Clock Plausibility Decision and full
B1c1 integration, a proposal must supply new paths, ownership, forecast, and
tests before writing code.

Later tests must cover activation/clock/credential prerequisites; fresh epoch
per reconnect and uint64 preservation; sequence/replay/gap/overflow,
replacement/stale close/cleanup/expiry; audit-event versus durable-audit
boundary; and no business behavior before ACTIVE. No runtime test is created
by this documentation-only umbrella.

## Remaining blockers

| Blocker | Required closure before a runtime candidate |
| --- | --- |
| Local state | Bounded continuity is decided: an already verified process retains only pre-existing local activity; unavailable, corrupt, or inconsistent SQLite forbids new sessions/authorizations, role/privilege changes, and cloud synchronization, while restart or absent verified in-memory state blocks authorization-dependent starts until recoverable integrity is restored. Existing security gates remain enforced; clear operator notification and detection/recovery recording where available are required. Independently verify this boundary before a runtime candidate. |
| Protected store | CLOSED as a spike (T25, L46-L47, run `303513b5`): in a disposable Windows Sandbox a non-exportable ECDSA P-256 machine key in the Microsoft Software KSP with its certificate in `Cert:\LocalMachine\My`, signing through `NT SERVICE\InfoplazaSecureLink`, export refused for the service and an administrator, and non-admin/alternate identities denied (key-file ACL SYSTEM and Administrators full, service SID read, no Users/Everyone ACE). SYSTEM and local administrators can still use the key; no claim blocks them. Remaining before a runtime candidate: confirm on the target Windows builds and on a non-synthetic issuance path, and handle the TPM/PCP `0x80090010` NO-GO. |
| Reconciliation | CLOSED as a documented contract (accepted 2026-11-20 after two independent verifications), see `reconciliation-wire-contract.md` (generic registry, atomic versioned authorization snapshot first, immutable `operation_id`, accept-once results, FIFO per center without holding later records, authorization by creation version, quarantine only for conflicts, authorized reviewer role to be defined). Open for the runtime candidate: Enlace authentication/delegation, reviewer role and channel, human-resolver authorization in the cloud, station-to-former-center sync path, cloud transport binding. |
| Audit | Decide durable/central audit access, deletion, recovery ownership and delivery. |
| `sourceId` | Define its remaining semantic relationship to station identity. |
| PKI issuance | Produce production issuance proof for the resolved URI-SAN identity schema. |
| Dinamizador keys | Prove protected physical certificate/key provisioning. |
| Trust/delegation | Define trust-bootstrap and delegation distribution. |
| B1c composition | CLOSED: B1c1a2 and B1c1b are integrated on master, and the exception-path hardening of B1c1b (fail-closed catch plus tests, issue #27, PR #28 in `Admin-AIP/sistema-infoplazas-dinamizador`, merge commit `7a7382c`) was merged on 2026-11-20; wiring the composition to the gateway belongs to B1c2 activation work. |
| Production limits | CLOSED as a documented decision in `production-limits.md` (accepted 2026-11-20): authorized station quota 10, `maxConnections` 2 x the authorized station count, `maxPayload` 64 KiB, `pingIntervalMs` 15 000 ms, with permitted ranges; the Enlace or the Operador del Centro may authorize stations beyond the quota, only the Enlace may change the technical values. The configuration mechanism stays open for the runtime candidate. |
| Local audit sink | Implement and prove the production durable local `SecurityAuditSink`, separately from central audit governance. |
| Integration | Perform a fresh integrated-code inspection and prepare a bounded candidate. |

No existing B1c artifact authorizes activation.
