# PR-08B1c2 sourceId semantics

Status: ACCEPTED by the maintainer on 2026-11-21; documentation only. Authorizes no runtime implementation and no
`ACTIVE` behavior. Closes the `sourceId` row of `design.md` "Remaining blockers" and OPEN REQUIREMENT 5 of
`changes/session-start-login-usuario-pc/tasks.md`.

## Question

`sourceId` is carried in the `link.hello` claims as an untrusted `source_id`, stored in the registry record and compared
for exact equality for every role. No document said what it identifies, who assigns it, its format or what happens to it
on reimage and center change.

## Terms

* Sending program (programa emisor): the program that opens the secure link to the Dinamizador and sends its messages
  (hello, requests, information, answers). On a Usuario PC it is the Usuario PC agent, the single program installed on
  that PC for this purpose; on the Dinamizador it is the Dinamizador program itself.
* It is not the person who uses the PC, not the machine hardware, and not each internal process, thread or window of the
  program: however many internal parts a program has, they share one sending program and one `sourceId`.
* A second program on the same PC that opened its own link to the Dinamizador would be a second sending program; that is
  the case in which this decision must be revisited.

## Decision

`sourceId` identifies the sending program, and it is derived, never typed or invented by a peer:

| Endpoint | `sourceId` | Assigned |
| --- | --- | --- |
| Usuario PC station | equal to its `stationId` | by the registry when the station is enrolled, together with `stationId` |
| Dinamizador (stationless) | equal to its own `installationId` | by the registry when the Dinamizador is installed |

Reason: each Usuario PC has a single program that sends messages to the Dinamizador, so a separate identifier adds a
concept to generate, enroll and maintain without adding information. Deriving it keeps enrollment automatic and keeps the
wire protocol unchanged: the field stays in the hello claims and the Usuario PC agent needs no change.

## Consequences

* The binding does not change: the claim `source_id` must equal the registry value exactly, for every role, with no
  trimming, case folding or derivation on the wire. A mismatch is `IDENTITY_MISMATCH`.
* `sourceId` stays registry-derived and is not carried in the certificate; the certificate keeps only `installationId`.
* Reimage: the station keeps its `stationId` and therefore its `sourceId`, even though keys and credential are new.
* Center change: the station receives a new identity for the receiving center, so its `sourceId` changes with it; the
  former value stays in the former center's records and audit.
* Reinstalling a Dinamizador creates a new `installationId` and therefore a new `sourceId`.
* Grammar and uniqueness: the same grammar as `stationId` and `centerId` applies (`^[a-z0-9](?:[a-z0-9._-]{0,62}[a-z0-9])?$`),
  which an `installationId` UUID in lowercase satisfies, and the registry rejects at ingestion a `sourceId` that is
  already used by another record.
* Reconciliation: `sourceId` is not part of the reconciliation contract, which keeps using `operation_id` and
  `created_under_version`. It stays an optional field of security audit events, as today.

## Revisit when

A single Usuario PC gets more than one program that sends messages to the Dinamizador, or when a message origin finer
than the station is needed. Then an independent identifier with its own lifecycle must be decided.

## Not decided here

The registry ingestion implementation and schema; aligning the legacy `promoteStationIdentity` path, which is stationful
only and treats `sourceId` as optional, with the rule that `sourceId` is mandatory for every role; any Usuario PC
conclusion; runtime implementation.
