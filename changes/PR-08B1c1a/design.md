# PR-08B1c1a Envelope Codec + Gateway Application Handoff

## Status: NOT STARTED — approved preimplementation design only

## Decision

B1c1a supplies the missing Dinamizador-side PR-07 Agent v2 outer-envelope
codec and delivers only a decoded, untrusted `link.hello` to a post-mTLS
application handoff. It preserves current TLS 1.3 mTLS, date validation,
resumption rejection, WSS upgrade, heartbeat, and cleanup. It follows
[B1b2b2](../PR-08B1b2b2/design.md), does not authorize, and is consumed by
[B1c1b](../PR-08B1c1b/design.md).

## Exact decoding boundary

B1c1a implements the server half of the PR-07 outer contract
([`design.md:190-229`](../session-start-login-usuario-pc/design.md#82-sobre-agent-v2-cerrado-para-pr-07)).
No server codec exists in the inspected Dinamizador tree; its only transport
implementation is `apps/desktop/electron/security/transport/tls-gateway.ts`.

| Area | Required rule |
| --- | --- |
| Outer envelope | JSON object; `protocol_version: 2`, `payload_schema_version: 1`, usable `message_id`, `kind: request`, `type: link.hello`, RFC3339 `sent_at`, and payload |
| Claims | wire `source_id`, `station_id`, `center_id` are nonblank opaque strings mapped without normalization to existing camelCase `UntrustedPeerClaims` (`identity.ts:8-13`) |
| Exact hello | payload exactly `{build_version, capabilities}`; nonblank build version; nonempty, unique string capabilities including `secure_link_v1` |
| Prohibited metadata | no `connection_epoch`, `sequence`, `link_id`, or `idempotency_key` |
| Unknown fields | outer compatibility only: never alters validation/routing/privilege and never permits forbidden hello metadata/payload keys |

A v2 candidate never falls back to legacy parsing. Malformed/unusable input
fails closed; protocol/schema incompatibility never downgrades. Decoded claims
are response-correlation metadata and later comparison inputs, never registry
selection or authority.

## Gateway handoff

`TlsGatewayConfig` has no application callback (`tls-gateway.ts:17-28`). Its
upgrade handler obtains `request.socket` as a live `TLSSocket`, date-checks
`getPeerCertificate()`, and emits WSS at `tls-gateway.ts:188-234`.

**PROPOSED, not implemented:** an optional minimal
`TlsGatewayConfig.onApplicationConnection` callback after that safe upgrade.
It retains the accepted `WebSocket`, request, and authenticated `TLSSocket` (or
equivalently narrow read-only context), preserving correlation and allowing the
existing `extractPeerCertificateIdentity()` adapter, which expects `authorized`
and `getPeerX509Certificate()` (`certificate-identity.ts:38-75`).

The seam must retain non-resumed/authorized TLS, date and HTTP/WSS validation
before callback invocation; it must not copy raw certificates or fabricate
identity. A valid hello retains its original `message_id`; malformed/unusable
application input terminates and cleans up through existing gateway tracking.
Existing local event/audit may be invoked only at its existing seam and is not a
durable B2 claim. TLS failures are never JSON replies.

## Exclusions

B1c1a has no final authorization, deny/registry/capability work, final accept,
epoch, ACTIVE, clock, lifecycle/sequence/replay, durable audit, or product work
(**ZERO** product actions). A generic PR-07 encoder is not authorization
permission and cannot construct final `link.accept`. Current epoch validation is
`bigint`/`UINT64_MAX` (`connection-epoch.ts:3-27`), but no inspected exact JSON
uint64 serialization interface exists; B1c1a proposes none and must not use
lossy JavaScript `Number`.

## Planned surfaces, forecast, and future tests

Revalidate before writing:

- **PROPOSED new:** `apps/desktop/electron/security/transport/agent-v2-envelope-codec.ts`
  and corresponding `apps/desktop/test/main/security/transport/agent-v2-envelope-codec.test.ts`.
- **Existing:** `tls-gateway.ts` and `test/main/security/transport/tls-gateway.test.ts`
  for the callback/retention/cleanup seam. `identity.ts` and
  `certificate-identity.ts` are consumers, not B1c1a edits.

Contingent estimate: new production `+72/-0`, existing `+22/-6`, tests
`+130/-0`, total `+224/-6`, **NET 218**, **CHURN 230**. It is below 300 because
scope ends at decoder/handoff; stop for a split if reforecast exceeds 300.

Future tests own outer/inner exactness, snake/camel preservation,
malformed/protocol/schema/unknown-versus-forbidden fields, correlation,
certificate/socket retention, unchanged TLS security/close cleanup, and proof
of no auth/epoch/accept/ACTIVE/clock/replay/product behavior. This docs-only
slice creates no runtime tests.
