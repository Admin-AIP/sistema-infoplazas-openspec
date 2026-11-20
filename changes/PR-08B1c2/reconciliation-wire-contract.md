# PR-08B1c2 Reconciliation wire contract (accepted)

Status: ACCEPTED by the user on 2026-11-20 after two independent verifications (no blockers); documentation only. Authorizes no runtime implementation, no `ACTIVE`, no product behavior.
Closes the `Reconciliation` row of `design.md` "Remaining blockers".
Related: `design.md` in this folder (Remaining blockers) and the secure-link specification
(`changes/session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md`). Decisions below marked
**[user]** were chosen by the maintainer on 2026-11-19/20, those marked **[default]** are architect defaults accepted by
the maintainer on 2026-11-19.

## Glossary

* Dinamizador: the program.
* Operador del Centro (Operador): the person who operates the Infoplaza or center and uses Dinamizador.
* Enlace: the human role that installs the programs; it has greater scope and privileges than the Operador del Centro.

Design principle: the programs are easy to operate, resolve technical problems themselves where possible, and use human
review only for exceptions that need it; the reviewer and place are decided later.

## 1. Scope

The contract fixes the envelope usage, the authorization-first step, `operation_id` semantics, the confirmation result,
ordering, retry and quarantine for cloud reconciliation after local isolation. **[user]** It is a generic registry:
concrete operation types (for example session facts) are registered by later PRs (PR-14) and must reuse this contract
unchanged. No schema for any business payload is defined here.

## 2. Envelope and naming

* Message shape follows the Agent v2 JSON envelope conventions (`protocol_version=2`, `payload_schema_version=1`,
  `message_id`, `kind`, `type`, `sent_at`, `payload`; `response`/`ack` carry `correlation_id`). No new envelope, no
  downgrade, unknown fields inert. `operation_result` is a `response` and carries the `correlation_id` of the
  `operations_submit` it answers; `quarantine_resolve` and `authorization_snapshot` are cloud-initiated requests.
* Wire names are snake_case. Proposed `type` values, subject to the registered type catalog: `reconciliation.authorization_snapshot`
  (cloud to Dinamizador), `reconciliation.operations_submit` (Dinamizador to cloud), `reconciliation.operation_result`
  (cloud to Dinamizador), `reconciliation.quarantine_resolve` (cloud to Dinamizador).
* Transport scope: reconciliation runs between Dinamizador and the cloud over a mutually authenticated channel. The
  cloud transport binding (connection lifecycle, epoch/sequence or equivalent replay protection) is not defined here.
  The station-link `ACTIVE` state, `connection_epoch` and `sequence` rules belong to the Dinamizador to Usuario PC link
  and do not apply to this traffic; reconciliation never depends on a station being connected.
  `message_id` stays transport deduplication; it is never business idempotency.
* `sent_at` stays observational. Sensitive payloads never enter security audit; audit keeps its finite vocabularies.

## 3. Authorization first **[user]**

1. On reconnection the cloud sends one `authorization_snapshot` containing a monotonically increasing
   `authorization_version` and the complete allow/deny state for the center, installation and station.
2. Dinamizador replaces the authorization/deny tables **atomically** with that snapshot, recording the version.
3. Only after the replacement commits may queued records be prepared and submitted. If the snapshot is rejected,
   incomplete, older than the stored version, or the replacement fails, nothing is prepared or submitted (fail closed)
   and the isolation rules of the degraded-state policy stay in force.
4. A snapshot with the same version as the stored one performs no replacement (idempotent retry); preparation and
   submission may proceed because the stored state is that version.
5. **[user]** A record without a valid `created_under_version` is never submitted and is quarantined locally with
   `reason_code` `invalid_version`.

## 4. `operation_id` and creation version

* Created once, when the record is queued; immutable; never regenerated, reused for another record, or derived from
  the retry count. A retry always resends the same `operation_id`.
* Each queued record carries `created_under_version`: the `authorization_version` in force in Dinamizador SQLite when
  the record was created. It is distinct from the submit `authorization_version`. Only Dinamizador writes it, from its own
  SQLite: it is never supplied by Usuario PC and never copied from a cloud message.
* It is the sole business idempotency identity of reconciliation. **[default]** If the envelope carries
  `idempotency_key` it must equal `operation_id`; reconciliation does not require it.
* Loss of a confirmation is never permission to create a new operation.

## 5. Submit and confirmation

* `operations_submit` carries a bounded batch of queued records, each with `operation_id`, operation `type`, the record
  reference, `created_under_version`, and the minimal sync payload already allowed by the SQLite queue schema (no keys,
  user data or unrestricted business payloads). It also carries the `authorization_version` of the authorization snapshot
  step against which the batch was prepared; this is not `created_under_version`. **[user]** The cloud rejects a
  record whose `created_under_version` is unknown, in the future, or outside the version history of that installation
  (version scope is per installation); such a `rejected` result carries a finite `reason_code` and is audited.
* The cloud evaluates whether the actor and center were authorized in the record's `created_under_version`. If they were,
  it accepts the record even if authorization was later revoked or the station later changed center. It returns that record
  in `operation_result` with a finite `flags` list containing `revoked_after` and/or `center_changed_after` when applicable.
  No signed time or local clock is used for this decision. Dinamizador may have had stale authorization when it created a
  record; that risk is accepted by the SQLite continuity policy, under which local SQLite is the sole authority during
  isolation.
* The cloud answers per operation in `operation_result` with a finite `result`: `accepted`, `already_received`,
  `rejected` (with a finite `reason_code`) or `quarantine_required` (with `reason_code` `conflict`). **[default]** Each
  result also returns the cloud's current `authorization_version`; a higher version than the one submitted means the
  snapshot step must run again before further submission; records already submitted under an older version are answered
  under the rules above and are never executed twice.
* The cloud accepts a given `operation_id` once: later submissions return `already_received` without re-executing it.
* A record is marked delivered only after `accepted` or `already_received`. `rejected` is terminal and kept for audit **[user]**. A submit batch is bounded; the size is a runtime value.
* **[user]** Records quarantined locally (`invalid_version`, section 3 item 5) are reported to the cloud in the next
  `operations_submit` as a list of `operation_id` with `reason_code`, without payload, so an authorized reviewer has a source.

## 6. Ordering **[user]**

FIFO per center: records of one center are submitted and confirmed in queue order, one batch in flight per center.
Different centers proceed independently. A quarantined record is set aside; later records of that center continue in FIFO
order. A future operation type that needs strict order declares `requires_order` when registered; its records are held
behind a quarantined record. **[default]** A non-conflicting record for another center is never held.

## 7. Retry **[default]**

* Transport and partial failures are retryable with the same `operation_id`, bounded exponential backoff with jitter,
  and no attempt deadline. Limits and backoff values are configuration defaults, not security decisions, and must be set
  when the runtime candidate is prepared; the contract only forbids unbounded tight retry loops.
* `rejected` and `quarantine_required` are never retried automatically **[user]**.
* Authorization refresh success with a partially processed queue: confirmed records stay confirmed; unconfirmed records
  resume in order; conflicting records are quarantined; the queue is not paused as a whole.

## 8. Quarantine **[user]**

* Quarantine is reserved for a cloud-reported `quarantine_required` conflict and the local fail-closed
  `invalid_version` case in section 3 item 5. A quarantined record becomes an immutable local case. Storage stays inside
  the existing SQLite limits: the queue row keeps `operation_id`, record reference, state `quarantined` and `reason_code`;
  detection time and the last `authorization_version` are written to the audit table; the original record is retained where
  it already lives and is referenced, never copied into the queue. It is never deleted, edited or silently uploaded.
* Only the cloud authority resolves it, through `quarantine_resolve` naming the `operation_id` and a finite
  `resolution` **[user]**: `release` or `discard`. The cloud issues it after review by an authorized reviewer; the
  reviewer role and review channel are to be defined later. Non-quarantined records need no review.
  * `release` does not resubmit directly: it returns the case to the queue head of its center and the record must pass
    the section 3 authorization-first step and the section 6 ordering again, with the same `operation_id`. If it has no
    valid `created_under_version`, it is quarantined again; `discard` is then the only useful outcome, because the
    version is immutable.
  * `discard` is terminal; the case and its audit entries are retained, never deleted.
  * `quarantine_resolve` naming an unknown `operation_id`, a record that is not quarantined, one already accepted or
    terminal, or one of another center is rejected without changing stored state or counters.
* When records are quarantined, Dinamizador gives the Operador a plain-language notice with the count, that there is
  nothing for them to do, and that the case was reported to the cloud or will be reported in the next submit when detected
  locally. The Operador has no replay or resolution authority. **[user]** `quarantine_resolve` is authenticated by the
  mutually authenticated cloud channel only; no additional per-command signature is introduced.

Note: the secure-link specification and `design.md` were amended so that the quarantine reviewer is an
authorized reviewer with role and channel to be defined; the Enlace is not the default reviewer and nothing in this
contract names it as one. The mutually
authenticated channel authenticates the cloud, not the human who decides; the cloud must verify that whoever resolves a
case is authorized. The mechanism is not defined here.

## 9. Invariants kept

Fail closed; no sync before a successful authorization step; cloud authoritative for authorization, deny and center
changes after reconnection; SQLite is the sole local authorization/deny source during isolation; no private keys or user
data in the queue; every rejection leaves stored state and counters unchanged; no existing B1c artifact authorizes
activation.

## 10. Not decided here

Business payload schemas per operation type; numeric retry, batch-size and backoff values; central audit governance;
the exact `type` strings if the catalog already assigns others; the Enlace authentication/delegation mechanism; the
reviewer role and review channel; the mechanism by which the cloud verifies human-resolver authorization; the cloud
transport binding and replay protection; the channel and authorization-first path of the station synchronization to
the former center on a center change (secure-link spec); whether a cloud-rejected record with an unknown version needs a
human review path; any Usuario PC conclusion; runtime implementation.
