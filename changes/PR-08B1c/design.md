# PR-08B1c Secure-Link Delivery Reconciliation

## Status: NOT STARTED — design approved; implementation not authorized

## Decision

The former B1c delivery unit is superseded **for delivery** by two independent,
non-active slices:

```text
B1b2b2 COMPLETE + integrated
  -> B1c1a Envelope Codec + Gateway Application Handoff
  -> B1c1b Authorization + Negotiation Composition
  -> mandatory Clock Plausibility architecture decision
  -> B1c2 ACTIVE / lifecycle umbrella
  -> security/lifecycle-cohesive re-split after integrated-code reinspection
  -> separately authorized implementation
```

This documentation-only reconciliation neither creates a legacy B1c file nor
authorizes runtime work. [B1b2b2](../PR-08B1b2b2/design.md) remains the
completed authority: `authorizePeer()` is the sole normal producer of the
seven-field `AuthorizedPeerIdentity`.

## Review path

| Slice | State | Produces | Never produces |
| --- | --- | --- | --- |
| [B1c1a](../PR-08B1c1a/design.md) | NOT STARTED | decoded/routed untrusted hello and safe TLS/WSS application handoff | authority, final accept, epoch, ACTIVE |
| [B1c1b](../PR-08B1c1b/design.md) | NOT STARTED | authorized/negotiated/accept-ready **non-active** internal result | `link.accept`, epoch, product eligibility |
| [B1c2](../PR-08B1c2/design.md) | BLOCKED / planning | later lifecycle decision and cohesive split | an implementation candidate |

## Normative boundaries

The protocol authority is
[`design.md` §8.2–8.5](../session-start-login-usuario-pc/design.md#82-sobre-agent-v2-cerrado-para-pr-07)
and [`spec.md:255-445`](../session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md).
The outer Agent v2 envelope is `protocol_version=2`,
`payload_schema_version=1`, `message_id`, `kind`, `type`, `sent_at`, and
`payload`, with conditional `correlation_id` and optional claims. Unknown
outer fields are compatible only when they do not alter validation, routing, or
privilege; they do not permit forbidden hello metadata or an inexact payload.
Wire fields remain snake_case even when TypeScript contracts are camelCase.

`link.hello` is an exact request with `source_id`, `station_id`, `center_id`,
and `{build_version, capabilities}`; capabilities are nonempty, unique, and
include `secure_link_v1`. It has no `connection_epoch`, `sequence`, `link_id`,
or `idempotency_key`. The normative final correlated `link.accept` has a fresh,
random, nonzero `uint64` epoch and is the Usuario PC ACTIVE boundary. It belongs
only to an activation-capable B1c2 descendant. `link.reject` is post-mTLS and
usable/correlatable-hello only, with no epoch/sequence/ACTIVE state and only
the existing closed `RejectionCode` namespace.

## Security invariants

- B1c1a preserves claims as `UntrustedPeerClaims` inputs; parsing never grants
  authority. B1c1b invokes deny-first `authorizePeer()` and retains B1b2b2’s
  universal exact sourceId binding.
- Neither B1c1 slice constructs, serializes, emits, reserves, or precomputes a
  final accept/epoch; neither transitions ACTIVE or confers product eligibility.
  Product actions are **ZERO**.
- TLS failures stay transport/local-audit behavior, never JSON reject.
  Installation/credential deny closes without JSON or invented
  `AUTHORIZATION_DENIED`. Events do not claim durable B2 persistence.
- No certificate, private key, secret, or wholesale internal identity is
  serialized. `TrustedStationIdentity`/`promoteStationIdentity()` stay legacy.

The existing FSM only permits an ACTIVE target from its current context
(`apps/desktop/electron/security/secure-link/fsm.ts:8-79`); it does not force a
B1c1 transition, and no legacy-FSM refactor is authorized.

## Open gates and contingent forecast

Clock plausibility is OPEN. It does not block B1c1 parsing/handoff/
authorization/version/capability work, but blocks final accept generation/use/
emission, either ACTIVE transition, and production acceptance. Its architecture
decision must precede B1c2 implementation. `sourceId` semantics, durable audit,
production PKI, and protected server-cert provisioning remain open/later.

Planning estimates below are not measured diffs. Future writers reforecast
against integrated B1c1 code; if either B1c1 candidate exceeds 300 NET, stop
for a security-cohesive split. B1c1a+b are never a combined candidate. The
separate future actual gate is <=400 NET.

| Candidate | New production | Existing production | Tests | Total additions/deletions | NET | CHURN | Planning result |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| B1c1a | +72/-0 | +22/-6 | +130/-0 | +224/-6 | 218 | 230 | PASS |
| B1c1b | +96/-0 | +16/-4 | +128/-0 | +240/-4 | 236 | 244 | PASS |
| B1c2 umbrella | +145/-0 | +35/-8 | +240/-0 | +420/-8 | 412 | 428 | NOT VIABLE |

B1c1a retains its forecast: the server codec and narrow gateway seam remain
absent. B1c1b falls from +258/-4 by removing prohibited final-accept/epoch work
and tests. B1c2 is deliberately not mechanically split; clock architecture and
integrated-code inspection must first identify cohesive lifecycle seams.
