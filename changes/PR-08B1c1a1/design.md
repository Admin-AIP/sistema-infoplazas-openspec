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

Forecast: **+263/-0**, **NET 263**, **CHURN 263**. This passes the <=300
planning checkpoint with **37 NET** headroom. A fresh exact checkpoint is
required before implementation. If it is over 300 NET, stop and propose another
cohesive split; never weaken validation.

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

## Timestamp and compatibility boundary

`sent_at` is the exact existing PR-07 timestamp contract: RFC3339/
RFC3339Nano-compatible. Validation must check syntax, a real calendar
date/time, timezone/offset validity, required timezone, and normative
fractional seconds. Ad-hoc `Date.parse`-only acceptance is explicitly
insufficient. This is a **format** boundary only; B1c1a1 does not decide clock
plausibility.

Outer unknown fields are allowed only where normative compatibility permits.
They cannot affect routing, canonical fields, claims, validation, privilege, or
prohibited metadata. Unknown payload keys are rejected.

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
