# ADR: CLOCK_EVIDENCE audit contract

## Status: PROPOSED — institutional approval required

This ADR proposes the minimum local audit contract that a future B1c2a3a
implementation could use for Clock Evidence decisions. It changes no normative
requirement and authorizes no code, durable sink, platform adapter, recovery,
`link.accept`, `ACTIVE`, epoch, admission, session activation, or wire event.
Until the decisions marked **PENDING** receive explicit institutional approval,
this document is not an implementation authorization.

## Review path

1. Confirm that the existing audit wire contract stays unchanged.
2. Choose the institutional decisions in [Decision matrix](#decision-matrix).
3. Incorporate only approved choices into a later normative OpenSpec change.
4. Re-inspect integrated code before proposing B1c2a3a.

## Context

PR #22 merged a bounded, non-activating Clock Evidence admission refinement at
`e4f35a086a9794e43d1d49619380d1134e70fac2`. OpenSpec PR #5 merged its
reconciliation at `b5ca2599398d94c726397fc7ae1dac93a9587569`. The integrated
B1c2 design records that no independent B1c2a3 / `CLOCK_EVIDENCE` audit
delivery evidence exists yet.

The current normative secure-link audit contract has six exact categories,
four control results, and delivery states `RECORDED` and `UNAVAILABLE`.
`REJECTED` carries one of the fifteen existing protocol `RejectionCode` values.
It intentionally has no CLOCK_EVIDENCE category, local reason vocabulary,
proof-data rule, or audit replay contract.

## Scope and non-goals

**In scope:** an institutional proposal for a local-only audit taxonomy,
minimized fields, durable-delivery semantics, idempotency, and the audit
boundary around Clock Evidence decisions.

**Out of scope:** modifying `SecurityAuditSink`; selecting a Windows storage
adapter; retention implementation; provisioning verification; certificate
rotation/revocation; time authority; automatic recovery; wire protocol; and
all lifecycle or product behavior.

Audit timestamps remain observations of the local system clock. They neither
establish institutional time nor repair, extend, or reinterpret X.509 validity.

## Existing approved contract

| Area | Status | Current contract |
| --- | --- | --- |
| Wire rejection vocabulary | **APPROVED** | Exactly 15 `RejectionCode` values; no new code is proposed. |
| Audit categories | **APPROVED** | Exactly `AUTHENTICATION`, `IDENTITY_BINDING`, `NEGOTIATION`, `REPLAY_PROTECTION`, `LINK_LIFECYCLE`, and `PRIVILEGE_ELIGIBILITY`. |
| Control results | **APPROVED** | Exactly `SUCCESS`, `REJECTED`, `INVALIDATED`, and `FAILED`. |
| Delivery states | **APPROVED** | Exactly `RECORDED` or `UNAVAILABLE`; delivery is distinct from the control result. |
| Baseline minimization | **APPROVED** | No generic metadata/context bags; no secrets, credentials, private keys, raw frames, payloads, sensitive PII, or disability data. |
| Durable sink intent | **APPROVED** | A durable, bounded local sink with atomic rotation and rate control is normative; a production implementation is not demonstrated. |
| Clock Evidence audit mapping | **PENDING** | No integrated B1c2a3 mapping, local category, or delivery proof exists. |

## Decision matrix

| Decision | Status | Proposed technical position | Institutional decision required |
| --- | --- | --- | --- |
| Local category | **PROPOSED** | Add closed local-only `CLOCK_EVIDENCE`; it is never a wire type or `RejectionCode`. | Approve the category and its owner. |
| Local action | **PROPOSED** | Closed actions: `DECISION`, `PROVISIONING_ATTEMPT`, `EVIDENCE_MUTATION`, `MANUAL_RECOVERY_ATTEMPT`. | Confirm that these are the complete auditable actions. |
| Local reason | **PROPOSED** | Closed reasons: `INITIAL_PROVISIONED`, `ROLLBACK_DETECTED`, `EVIDENCE_UNAVAILABLE`, `EVIDENCE_CORRUPT`, `EVIDENCE_INDETERMINATE`, `PROVISIONING_VERIFIER_UNAVAILABLE`. | Approve semantics and whether additional reasons are necessary. |
| Result mapping | **PROPOSED** | `SUCCESS` only for a completed, verifiable local action; `REJECTED` for controlled evidence denial; `FAILED` for technical failure; never use `INVALIDATED` unless an existing link is actually invalidated by an approved later boundary. | Approve mappings per action. |
| Proof and provenance | **PENDING** | Do not record `proofId`, proof bytes, signer material, certificates, certificate fingerprints, or raw provenance. Record at most a closed non-secret verification class. | Decide whether a privacy-preserving correlation reference is ever justified. |
| Delivery meaning | **PROPOSED** | `RECORDED` means the allowlisted event has durably committed, or an identical durable event was found idempotently. `UNAVAILABLE` means that outcome cannot be proven. | Approve the operational meaning and evidence. |
| Persistence | **PENDING** | The future sink should be append-only or equivalent, bounded, atomically rotated, and access-controlled. This ADR selects no Windows mechanism. | Select retention, access, export, and operating owner. |
| Idempotency/replay | **PROPOSED** | Deduplicate by durable `eventId` plus a canonical hash of allowlisted content. Same ID/same hash is idempotent; same ID/different hash is a collision and is never overwritten. | Approve collision handling, alerting, and retention horizon. |
| Recovery | **PENDING** | Audit a manual recovery attempt only after an independently authorized recovery procedure exists. No automated repair or evidence overwrite is proposed. | Define recovery authority and evidence. |

## Proposed taxonomy

The following is a proposal, not a normative vocabulary:

| Action | Allowed result/reason relationship | Exclusions |
| --- | --- | --- |
| `DECISION` | `SUCCESS` only for a locally valid final decision; `REJECTED` for rollback, corruption, indeterminate evidence, or controlled denial; `FAILED` for unavailable storage/verifier or another technical failure. | Does not authorize admission or describe a wire rejection. |
| `PROVISIONING_ATTEMPT` | `SUCCESS` only after an approved proof verification and durable evidence commit; `REJECTED` when proof/binding validation fails; `FAILED` when the verifier or storage is unavailable. | Does not retain proof material or create authority. |
| `EVIDENCE_MUTATION` | `SUCCESS` only after the durable mutation commits; `FAILED` otherwise. | Never logs a proposed-but-uncommitted write as success. |
| `MANUAL_RECOVERY_ATTEMPT` | No mapping is authorized until recovery authority is approved. | No automatic recovery, cleanup-as-recovery, or overwrite. |

A category event must contain one closed action and one closed reason where the
action/result mapping requires it. `CERTIFICATE_INVALID` is intentionally not a
CLOCK_EVIDENCE reason: current X.509 failure remains independently represented
by the existing authentication contract. A CLOCK_EVIDENCE event must not use
`RejectionCode`, `transportRejectionReason`, arbitrary metadata, or a generic
error string.

## Separation from protocol rejection

`CLOCK_EVIDENCE` is proposed solely as a local audit concern. The existing
fifteen `RejectionCode` values remain exact and unchanged. A Clock Evidence
decision must not produce `link.reject`, extend a wire schema, or reinterpret a
protocol rejection. The existing `SecurityAuditSink` contract is not modified
by this ADR.

## Proposed allowlist and privacy boundary

A later approved contract may allow only:

- generated `eventId`;
- observational audit timestamp;
- local category, action, result, and closed reason;
- trusted `installationId` only when already known by existing trusted paths;
- protocol/schema version only when relevant to the local decision;
- Clock Evidence record schema version and decision schema version; and
- delivery status, recorded separately from the control result.

It must prohibit proof IDs, raw proof bytes, provenance strings, certificate
material or fingerprints, cryptographic material, credentials, keys, tokens,
raw frames, payloads, generic metadata/context, user/session content, PII, and
wall-clock history/high-water values. A timestamp is not a trusted-time claim.

## Proposed delivery, durability, and replay semantics

The proposed future durable sink contract is deliberately stricter than a
best-effort callback:

1. The event is structurally allowlisted before persistence.
2. A new `eventId` is committed together with the canonical hash of only
   allowed fields.
3. Returning `RECORDED` requires a durable commit, or discovery of the same ID
   with the same canonical hash.
4. Returning `UNAVAILABLE` covers an absent sink, write failure, unreadable
   dedupe state, or inability to prove a prior matching commit.
5. An existing ID with a different canonical hash is a collision/tamper
   condition: it must not overwrite either event or become `RECORDED`.
6. Bounded retention, atomic rotation, rate control, access control, export,
   and alerting remain institutional/operational decisions.

This proposal does not define a grace period, retry loop, automatic recovery,
or an operational storage implementation.

## Alternatives considered

| Alternative | Advantage | Risk / reason not recommended |
| --- | --- | --- |
| Reuse `AUTHENTICATION` or an existing `RejectionCode` | Lowest apparent code change | Misstates meaning and changes the closed protocol-audit relationship. |
| Generic audit metadata | Flexible | Violates the approved allowlist and makes privacy review unverifiable. |
| Best-effort fire-and-forget audit | Simple | Cannot establish `RECORDED`, durable delivery, or replay-safe history. |
| Local closed category and discriminated fields | Explicit, reviewable, wire-independent | Requires institutional agreement before any implementation. |

## Institutional responsibilities pending

The institution must explicitly decide:

1. the owner and complete vocabulary of CLOCK_EVIDENCE events;
2. whether any proof/provenance correlation is allowed and, if so, its privacy
   basis, keying, retention, and access controls;
3. the meaning of durable commit, duplicate, collision, and alert;
4. retention, access, export, incident response, and operating ownership;
5. the authority and evidence required for manual recovery; and
6. the relationship of these events to certificate rotation, revocation, and
   offline operation.

## Preconditions for B1c2a3a

A future implementation proposal may start only after all of the following:

- this ADR, or a successor, is explicitly approved and reflected in a tracked
  normative OpenSpec contract;
- local taxonomy, allowlist, result mapping, delivery semantics, and replay
  behavior are approved;
- any provisioning/provenance and manual-recovery policy is approved or is
  explicitly excluded from the candidate;
- the durable sink boundary has an approved owner and verification plan; and
- a fresh inspection confirms the integrated contracts and still finds no
  activation, wire, session, or product scope in B1c2a3a.

## Consequences

This ADR preserves the current fail-closed security boundary without claiming a
new institutional authority. It intentionally delays B1c2a3a until a reviewer
can verify its audit semantics, privacy limits, and durable-delivery guarantee.
