# CLOCK_EVIDENCE: proposed local audit contract and institutional review

**Document identifier:** `PR-08-CLOCK-EVIDENCE-CONTRACT-REVIEW`

**Status:** PROPOSED — institutional approval required

**Purpose:** document-only OpenSpec change for review. It is deliberately not
`B1c2a3a`, does not authorize that future implementation, and does not change
any normative contract.

## Review outcome requested

Reviewers are asked to approve, reject, or amend the proposed minimum local
`CLOCK_EVIDENCE` audit contract below. Until each item marked **PENDING** has
an explicit institutional disposition, this document is not an implementation
authorization.

## Authority and classification

| Classification | Content in this document | Authority status |
| --- | --- | --- |
| A. Confirmed functional decisions | Local-only boundary; first-cut actions; exclusions; six exact reasons; safety non-interruption boundary. | Confirmed by the project owner; not a change to a normative contract. |
| B. Technical recommendations | Minimal allowlist, separation of control result from delivery state, durable/idempotent recording characteristics. | Technical recommendation only; not authorization from Security, Privacy, or IT Operations. |
| C. Proposed rules awaiting approval | The complete action/result/reason matrix, collision handling, and persistence semantics. | **PENDING** institutional approval. |
| D. Institutional matters undefined | Retention, access, export, deletion, operational owner, alert and incident process. | **PENDING** institutional definition. |

The original ADR remains unchanged at
[`changes/PR-08B1c2/adr-clock-evidence-audit-contract.md`](../PR-08B1c2/adr-clock-evidence-audit-contract.md).
This document refines its review material; it neither supersedes it nor makes
its proposals normative.

## A. Confirmed functional boundary

1. `CLOCK_EVIDENCE` is exclusively a **local** audit category. It is outside
   the wire protocol and outside the exact fifteen existing `RejectionCode`
   values.
2. The first cut permits only these local actions: `DECISION` and
   `EVIDENCE_MUTATION`.
3. `PROVISIONING_ATTEMPT` and `MANUAL_RECOVERY_ATTEMPT` are explicitly
   excluded from this change. No automatic or manual recovery is introduced.
4. The exact, closed reason vocabulary under review is:
   `INITIAL_PROVISIONED`, `ROLLBACK_DETECTED`, `EVIDENCE_UNAVAILABLE`,
   `EVIDENCE_CORRUPT`, `EVIDENCE_INDETERMINATE`, and
   `PROVISIONING_VERIFIER_UNAVAILABLE`.
5. Audit, network, or clock failures must never close an application or
   interrupt an existing student session.
6. This change must not activate `link.accept`, `ACTIVE`, `connection_epoch`,
   operational admission, or sessions.
7. This change does not modify `SecurityAuditSink`, `tls-gateway.ts`, code,
   a protocol, or any PR-08U1 material.

## B. Technical recommendations

### Minimal proposed allowlist

A future approved local event may contain only the following fields:

| Field | Proposed use |
| --- | --- |
| `eventId` | Generated local event identifier; idempotency key. |
| `observedAt` | Observational local audit timestamp; never a trusted-time assertion. |
| `category` | Closed value `CLOCK_EVIDENCE`. |
| `action` | Closed value `DECISION` or `EVIDENCE_MUTATION`. |
| `controlResult` | Closed value `SUCCESS`, `REJECTED`, or `FAILED`. |
| `reason` | One of the six closed reasons in this document. |
| `installationId` | Only when already available through an existing trusted local path. |
| `clockEvidenceSchemaVersion` | Closed schema/version identifier for the local record. |
| `decisionSchemaVersion` | Closed schema/version identifier when the action is `DECISION`. |
| `deliveryState` | `RECORDED` or `UNAVAILABLE`, stored separately from `controlResult`. |

The following are prohibited: `proofId`; proof bytes; certificates;
certificate material; public-key or certificate fingerprints; credentials,
keys, tokens, or cryptographic material; identity fingerprints; full evidence;
high-water or wall-clock history; PII; user/session content; payloads; raw
frames; generic error strings; free metadata/context; and arbitrary provenance.
No field outside the allowlist may participate in the canonical hash.

### Result and delivery are distinct

`controlResult` describes the local control outcome only:

- `SUCCESS`: the proposed action completed under the approved mapping;
- `REJECTED`: the action was deliberately and safely denied under the approved
  mapping; and
- `FAILED`: a technical condition prevented the action under the approved
  mapping.

`deliveryState` describes local audit persistence only:

- `RECORDED` is permitted only after a durable local persistence commit is
  proven, or an identical event was already durably persisted; and
- `UNAVAILABLE` applies whenever that proof cannot be made, including absent or
  unavailable storage, a failed write, unreadable deduplication state, or a
  collision.

An audit failure, delivery state, or timestamp observation must not alter the
control result after the fact and must not cause application closure or student
session interruption.

### Proposed durable and idempotent local recording

A future implementation should use a local, durable append-only store or a
security-equivalent mechanism with atomic rotation, access control, bounded
capacity, and rate limiting. This proposal selects **no** unevaluated Windows
adapter.

Idempotency is proposed as follows:

1. Construct a canonical hash from exactly the permitted fields, in a
   documented stable serialization. Excluded fields never enter the hash.
2. Persist a new `eventId` with that canonical hash.
3. A repeated `eventId` with the identical canonical hash may return
   `RECORDED` only when its prior durable record can be proven.
4. A repeated `eventId` with a different canonical hash is a collision. Do not
   overwrite either record and do not report `RECORDED`.
5. The alerting, incident classification, owner, and response for a collision
   are **PENDING** institutional decisions. The alternatives are a local
   operational alert, an incident workflow, both, or an explicitly approved
   non-alert handling policy.

## C. Proposed action–result–reason matrix (not approved)

Every row below is **PENDING**. “Recommended” identifies the technical
recommendation derived from the ADR; it is not an approved mapping. “Do not
permit” means this proposal does not authorize the combination. Each unresolved
row states its available institutional alternatives rather than silently
inventing an approval.

For `EVIDENCE_MUTATION`, the ADR fixes only the operation-result boundary:
`SUCCESS` requires a durable mutation commit; otherwise the operation result is
`FAILED`. It does not approve a reason-specific `FAILED` mapping. Consequently,
this matrix recommends no `REJECTED` result and leaves every reason-specific
`FAILED` row **PENDING**.

| Action | Control result | Reason | Proposed disposition | Pending decision / alternatives |
| --- | --- | --- | --- | --- |
| `DECISION` | `SUCCESS` | `INITIAL_PROVISIONED` | PENDING — recommended only after a valid final local decision. | Approve this mapping, select another closed success reason, or forbid success events for this reason. |
| `DECISION` | `SUCCESS` | `ROLLBACK_DETECTED` | PENDING — do not permit. | Keep prohibited, or define a distinct approved success semantic. |
| `DECISION` | `SUCCESS` | `EVIDENCE_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, or redefine availability semantics institutionally. |
| `DECISION` | `SUCCESS` | `EVIDENCE_CORRUPT` | PENDING — do not permit. | Keep prohibited, or define a distinct approved success semantic. |
| `DECISION` | `SUCCESS` | `EVIDENCE_INDETERMINATE` | PENDING — do not permit. | Keep prohibited, or define a distinct approved success semantic. |
| `DECISION` | `SUCCESS` | `PROVISIONING_VERIFIER_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, or redefine verifier-unavailability semantics institutionally. |
| `DECISION` | `REJECTED` | `INITIAL_PROVISIONED` | PENDING — do not permit. | Keep prohibited, or define a controlled-denial semantic. |
| `DECISION` | `REJECTED` | `ROLLBACK_DETECTED` | PENDING — recommended controlled denial. | Approve this mapping, use `FAILED`, or forbid the event. |
| `DECISION` | `REJECTED` | `EVIDENCE_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, use `FAILED`, or define a controlled-denial policy. |
| `DECISION` | `REJECTED` | `EVIDENCE_CORRUPT` | PENDING — recommended controlled denial. | Approve this mapping, use `FAILED`, or forbid the event. |
| `DECISION` | `REJECTED` | `EVIDENCE_INDETERMINATE` | PENDING — recommended controlled denial. | Approve this mapping, use `FAILED`, or forbid the event. |
| `DECISION` | `REJECTED` | `PROVISIONING_VERIFIER_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, use `FAILED`, or define a controlled-denial policy. |
| `DECISION` | `FAILED` | `INITIAL_PROVISIONED` | PENDING — do not permit. | Keep prohibited, or define a technical-failure semantic. |
| `DECISION` | `FAILED` | `ROLLBACK_DETECTED` | PENDING — do not permit. | Keep prohibited, use `REJECTED`, or define a technical-failure semantic. |
| `DECISION` | `FAILED` | `EVIDENCE_UNAVAILABLE` | PENDING — recommended technical failure. | Approve this mapping, use `REJECTED`, or forbid the event. |
| `DECISION` | `FAILED` | `EVIDENCE_CORRUPT` | PENDING — do not permit. | Keep prohibited, use `REJECTED`, or define a technical-failure semantic. |
| `DECISION` | `FAILED` | `EVIDENCE_INDETERMINATE` | PENDING — do not permit. | Keep prohibited, use `REJECTED`, or define a technical-failure semantic. |
| `DECISION` | `FAILED` | `PROVISIONING_VERIFIER_UNAVAILABLE` | PENDING — recommended technical failure. | Approve this mapping, use `REJECTED`, or forbid the event. |
| `EVIDENCE_MUTATION` | `SUCCESS` | `INITIAL_PROVISIONED` | PENDING — recommended only when an initial local record durably commits. | Approve this mapping, reserve it for `DECISION`, or define another closed success reason. |
| `EVIDENCE_MUTATION` | `SUCCESS` | `ROLLBACK_DETECTED` | PENDING — do not permit. | Keep prohibited, or define a distinct approved success semantic. |
| `EVIDENCE_MUTATION` | `SUCCESS` | `EVIDENCE_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, or redefine availability semantics institutionally. |
| `EVIDENCE_MUTATION` | `SUCCESS` | `EVIDENCE_CORRUPT` | PENDING — do not permit. | Keep prohibited, or define a distinct approved success semantic. |
| `EVIDENCE_MUTATION` | `SUCCESS` | `EVIDENCE_INDETERMINATE` | PENDING — do not permit. | Keep prohibited, or define a distinct approved success semantic. |
| `EVIDENCE_MUTATION` | `SUCCESS` | `PROVISIONING_VERIFIER_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, or redefine verifier-unavailability semantics institutionally. |
| `EVIDENCE_MUTATION` | `REJECTED` | `INITIAL_PROVISIONED` | PENDING — do not permit. | Keep prohibited, or define a separately approved action/result semantic. |
| `EVIDENCE_MUTATION` | `REJECTED` | `ROLLBACK_DETECTED` | PENDING — do not permit. | Keep prohibited, or define a separately approved action/result semantic. |
| `EVIDENCE_MUTATION` | `REJECTED` | `EVIDENCE_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, or define a separately approved action/result semantic. |
| `EVIDENCE_MUTATION` | `REJECTED` | `EVIDENCE_CORRUPT` | PENDING — do not permit. | Keep prohibited, or define a separately approved action/result semantic. |
| `EVIDENCE_MUTATION` | `REJECTED` | `EVIDENCE_INDETERMINATE` | PENDING — do not permit. | Keep prohibited, or define a separately approved action/result semantic. |
| `EVIDENCE_MUTATION` | `REJECTED` | `PROVISIONING_VERIFIER_UNAVAILABLE` | PENDING — do not permit. | Keep prohibited, or define a separately approved action/result semantic. |
| `EVIDENCE_MUTATION` | `FAILED` | `INITIAL_PROVISIONED` | PENDING — `FAILED` only when the mutation does not complete; no reason-specific mapping is recommended. | Keep this result boundary without approving this reason mapping, or define a different approved mapping. |
| `EVIDENCE_MUTATION` | `FAILED` | `ROLLBACK_DETECTED` | PENDING — `FAILED` only when the mutation does not complete; no reason-specific mapping is recommended. | Keep this result boundary without approving this reason mapping, or define a different approved mapping. |
| `EVIDENCE_MUTATION` | `FAILED` | `EVIDENCE_UNAVAILABLE` | PENDING — `FAILED` only when the mutation does not complete; no reason-specific mapping is recommended. | Keep this result boundary without approving this reason mapping, or define a different approved mapping. |
| `EVIDENCE_MUTATION` | `FAILED` | `EVIDENCE_CORRUPT` | PENDING — `FAILED` only when the mutation does not complete; no reason-specific mapping is recommended. | Keep this result boundary without approving this reason mapping, or define a different approved mapping. |
| `EVIDENCE_MUTATION` | `FAILED` | `EVIDENCE_INDETERMINATE` | PENDING — `FAILED` only when the mutation does not complete; no reason-specific mapping is recommended. | Keep this result boundary without approving this reason mapping, or define a different approved mapping. |
| `EVIDENCE_MUTATION` | `FAILED` | `PROVISIONING_VERIFIER_UNAVAILABLE` | PENDING — `FAILED` only when the mutation does not complete; no reason-specific mapping is recommended. | Keep this result boundary without approving this reason mapping, or define a different approved mapping. |

The matrix intentionally contains no rows for `PROVISIONING_ATTEMPT` or
`MANUAL_RECOVERY_ATTEMPT`; both are excluded from the first cut. It also does
not use `INVALIDATED`: that existing result remains outside this proposal unless
a separately approved boundary actually invalidates an existing link.

## D. Institutional matters still undefined

The following must receive an explicit institutional decision before a future
normative contract or implementation proposal:

1. minimum and maximum retention;
2. who may access local records and under what control;
3. whether, how, and by whom records may be exported;
4. deletion/rotation authority and verified deletion behavior;
5. operational owner for capacity, rate limits, monitoring, and incident
   response;
6. collision alerting and incident escalation;
7. the evidence for a durable commit and acceptable equivalent to append-only
   storage; and
8. whether any of the matrix rows marked PENDING becomes allowed, forbidden, or
   requires a differently scoped future action.

## Explicit non-goals and safeguards

- No recovery implementation, automatic recovery, manual recovery, evidence
  overwrite, cloud service, Supabase integration, or automatic telemetry.
- No wire event, no `RejectionCode`, no protocol change, and no change to the
  existing audit sink contract.
- No app shutdown, existing student-session interruption, admission change, or
  activation/lifecycle change due to audit, network, or clock failure.
- No selection of a Windows adapter that has not been evaluated and approved.
- No implementation work from B1c2a3a and no PR-08U1 change.

## Validation checklist for this document

- [x] The original ADR is preserved and referenced.
- [x] The proposed category is local-only and separated from the fifteen wire
  `RejectionCode` values.
- [x] Only `DECISION` and `EVIDENCE_MUTATION` appear in the matrix.
- [x] All six ADR reason strings appear exactly in every action/result matrix
  combination.
- [x] Every unapproved combination is explicitly marked **PENDING** with
  alternatives.
- [x] The allowlist and prohibited data boundary are explicit.
- [x] `controlResult` and `deliveryState` are distinct; `RECORDED` requires
  durable proof or a proven identical duplicate.
- [x] Collision behavior forbids overwrite and `RECORDED`.
- [x] No code, protocol, secret, or sensitive evidence is introduced.

## Next institutional step

Treat this document as a review artifact only. A future normative OpenSpec
change requires explicit institutional approval of the pending items above;
B1c2a3a remains **NO-GO** until then.
