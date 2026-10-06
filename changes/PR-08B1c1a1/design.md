# PR-07 Envelope Codec + Focused Validation

## Status: NOT STARTED — approved preimplementation design only

## Decision

B1c1a1 delivers only the Dinamizador-side PR-07 Agent v2 outer-envelope
decoder and its focused validation. It follows B1b2b2 and precedes B1c1a2. It
has **no** gateway callback or application handoff. If a meaningful gateway
production modification becomes necessary, stop; that work belongs to B1c1a2
or requires a new cohesive split.

## Planned surfaces and delivery gate

| Surface | Planned role |
| --- | --- |
| `apps/desktop/electron/security/transport/agent-v2-envelope-codec.ts` | production decoder |
| `apps/desktop/test/main/security/transport/agent-v2-envelope-codec.test.ts` | focused validation tests |

The prior **278-NET provisional forecast** is planning history only and is not
automatically promoted. B1c1a1 remains **NOT STARTED** and this contract closure
grants no implementation authorization. A **NEW fresh preimplementation size
checkpoint** is required after this closure and before any implementation; if it
is over 300 NET, stop and propose another cohesive split rather than weakening
validation. B1c1a2 and B1c1b remain **BLOCKED**; this closure does not alter
their plans or authorize their implementation.

## Exact decoding contract

A valid `link.hello` candidate is a JSON object with all of these exact
properties:

| Area | Required rule |
| --- | --- |
| Envelope | `protocol_version` exactly `2`; `payload_schema_version` exactly `1`; usable `message_id`; `kind` exactly `request`; `type` exactly `link.hello`; valid normative `sent_at`; `source_id`; `station_id`; `center_id`; and `payload` |
| Payload | exactly `build_version` and `capabilities`; `build_version` is a nonblank string; `capabilities` is a nonempty string array with no duplicates and includes `secure_link_v1` |
| Claims | map wire `source_id`, `station_id`, and `center_id` exactly into existing camelCase `UntrustedPeerClaims` |
| Correlation | preserve original usable `message_id` exactly; generate no replacement; if the generic envelope permits optional `correlation_id`, preserve it exactly and validate it to that existing contract without inventing hello correlation semantics |
| Prohibited hello metadata | reject `connection_epoch`, `sequence`, `link_id`, and `idempotency_key`; never silently discard them |

Claims remain **UNTRUSTED**. Their mapping applies no trim, case folding, Unicode
normalization, canonicalization, repair, fallback, or derivation. They do not
select a registry, grant authority, or affect privilege.

## Strict `sent_at` profile and compatibility boundary

`sent_at` uses the authoritative strict ADA PR-07 profile, not merely a
«RFC3339/RFC3339Nano-compatible» parser. The only accepted grammar is
`YYYY-MM-DDTHH:MM:SS[.fraction](Z|+HH:MM|-HH:MM)`.

- Date is exactly a four-digit year plus two-digit month/day. Month is `01`–`12`; the date MUST be a real Gregorian calendar date, including leap-year validation, and MUST NOT be repaired.
- The separator is uppercase `T` only; lowercase `t`, spaces, and alternatives are rejected. Time is two-digit `HH:MM:SS`: `HH` `00`–`23`, `MM` `00`–`59`, and `SS` `00`–`59`; 24-hour overflow and `SS=60` are rejected.
- Leap seconds are intentionally unsupported and rejected for deterministic cross-runtime behavior, no leap-second-table dependency, and because `sent_at` is observational while epoch/sequence own replay protection. No conditional leap-second handling exists.
- Fraction is optional. If present, it is a dot plus 1–9 decimal digits. Empty fraction, 10+ digits, and comma separator are rejected. Precision and trailing zeros are preserved exactly.
- Timezone is mandatory: uppercase `Z` or signed numeric `+HH:MM`/`-HH:MM`, with offset hour `00`–`23` and minute `00`–`59`. Accepted examples are `Z`, `+00:00`, `-05:00`, `+05:30`, and `+23:59`. Malformed offsets and `-00:00` are rejected; `-00:00` is not accepted as RFC3339's unknown-local-offset convention. Uppercase `T` and `Z` only: lowercase `t`/`z` are rejected.
- Validation preserves the original accepted string exactly. It MUST NOT convert to `Z`, change fractions, trim trailing zeros, uppercase, repair date/offset, or substitute a `Date` reserialization.

Ad-hoc `Date.parse`-only acceptance is explicitly insufficient. This closes only
format/calendar/timezone. **Clock Plausibility** remains OPEN and separate;
B1c1a1 MUST NOT implement it. It still blocks later activation-capable final
accept and `ACTIVE`.

Outer unknown fields are allowed only where normative compatibility permits.
They cannot alter routing, override canonical fields, create alternate claims,
bypass validation, grant privilege, or legalize prohibited hello metadata.
Unknown payload keys are rejected.

## Exclusions

This slice excludes gateway handoff/callbacks, live socket retention,
certificate extraction, registry/deny/`authorizePeer`, capability negotiation,
`link.reject`/`link.accept`, epoch, FSM/ACTIVE, replay/sequence/lifecycle,
durable audit, and product actions. Product actions are **ZERO**.

Clock Plausibility remains OPEN but does not block this codec/format work.
`sourceId` semantics also remain OPEN and are not reopened here. B1c1a1 is
complete only when its approved implementation is integrated; only then may
B1c1a2 begin.

## Future test ownership

Focused tests must prove exact envelope and payload acceptance, rejection of
malformed/protocol/schema/metadata/payload variants, claim preservation without
normalization, message/correlation preservation, and strict timestamp boundary
behavior. This documentation-only slice adds no runtime tests.
