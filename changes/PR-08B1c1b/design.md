# PR-08B1c1b Authorization + Negotiation Composition

## Status: NOT STARTED — approved preimplementation design only

## Decision

B1c1b consumes [B1c1a](../PR-08B1c1a/design.md), validates exact hello,
authorizes untrusted claims through B1b2b2, and negotiates existing
capabilities. Its success is internal, authorized/negotiated/accept-ready, and
**NON-ACTIVE**.

It MUST NOT construct any final normative `link.accept`, including “composed
but unemitted”; serialize, emit, reserve, or generate `connection_epoch`;
transition either peer ACTIVE; or confer product eligibility. Final accept and
a fresh epoch belong only at an activation-capable
[B1c2 descendant](../PR-08B1c2/design.md).

## Inputs, order, and result

B1c1b receives B1c1a’s decoded hello, usable `message_id` correlation, and
narrow live connection/certificate context. It maps wire `source_id`,
`station_id`, and `center_id` into existing camelCase `UntrustedPeerClaims`
(`identity.ts:8-13`). Protocol/schema/hello checks independent of authority
may precede authorization, but no claim binding or success precedes these APIs:

```text
B1c1a untrusted hello/application context
  -> existing certificate identity / registry resolution
  -> authorizePeer(registryResolved, untrustedClaims, denyRegistry)
  -> negotiateCapabilities(hello.capabilities)
  -> internal authorized/negotiated/non-active result
```

`authorizePeer()` (`claim-authorization.ts:25-37`) calls `evaluateDeny()` before
private exact binding and returns exactly `installationId`, `stationId`,
`centerId`, `sourceId`, `endpointRole`, `credentialFingerprint256`, and
`registryVersion` (`claim-authorization.ts:8-23`). B1c1b consumes it, never
legacy `TrustedStationIdentity`/`promoteStationIdentity()`; the latter has no
B1b certificate/registry/deny path ([B1b2b2](../PR-08B1b2b2/design.md#legacy-promotestationidentity-boundary)).

It calls current `negotiateCapabilities()` only. It allowlists the intersection,
requires `secure_link_v1`, treats malformed/empty/duplicate input as existing
`HANDSHAKE_MALFORMED`, and absence of the mandatory capability as existing
`CAPABILITY_REQUIRED` (`capabilities.ts:3-31`). Unknown offers grant zero
privilege.

The established result convention is `SecureLinkResult<T, E>`
(`contracts.ts:23-26`). After inspecting it and `LinkContext`, this proposes
— but does not implement — **`AuthorizedNegotiatedPeerContext`**: a non-wire
`SecureLinkResult` success carrying only the seven authority fields, exact
accepted versions, negotiated capabilities, usable hello/message correlation,
and narrow B1c1a connection/application/credential context. It is not a wire
DTO, `LinkAcceptPayload`, activation grant, or FSM replacement; it contains no
final envelope, epoch, sequence, link id, business command, or eligibility
flag. B1c1b does not write `LinkContext` or call its ACTIVE transition
(`fsm.ts:8-79`).

## Fail-closed response and close policy

Per [`spec.md:318-350`](../session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md),
a reject is post-mTLS and only for a usable/correlatable hello; it correlates
exactly, has no epoch/sequence/ACTIVE state, and uses only `contracts.ts:5-21`.

| Condition | Required action |
| --- | --- |
| identity/center mismatch | MAY safely correlate existing `IDENTITY_MISMATCH`/`CENTER_MISMATCH` reject, then close |
| malformed/unsupported/capability/timeout/out-of-scope | existing approved code only when safe to respond; otherwise close; never downgrade/invent payload |
| `INSTALLATION_DENIED`/`CREDENTIAL_DENIED` | MUST close/terminate: no JSON, success/ACTIVE, or `AUTHORIZATION_DENIED`; use existing local event/audit seam only where appropriate |
| registry failure/unexpected exception | fail closed and close; no invented wire/audit taxonomy |
| pre-application TLS failure | preserve transport/local-audit behavior; never JSON |

Event invocation is not durable B2 persistence. Never serialize certificates,
private keys, secrets, serial number, or wholesale internal identity/context.

## Exclusions, forecast, and future tests

B1c1b retains B1b2b2’s universal exact sourceId binding, including stationless
roles; sourceId semantics remain OPEN. It excludes final accept/epoch/uint64,
ACTIVE/clock, sequence/replay/gap/overflow, replacement/stale/expiry, durable
audit, registry/deny changes, legacy-FSM refactor, and business actions
(**ZERO** product actions).

Revalidate after B1c1a integration. **PROPOSED new:**
`apps/desktop/electron/security/secure-link/handshake-composition.ts` and its
corresponding test. Existing consumers are `claim-authorization.ts`,
`capabilities.ts`, `contracts.ts`, `certificate-identity.ts`,
`installation-registry.ts`, and B1c1a’s handoff; change only minimal types/
wiring, never `RejectionCode` or final-accept behavior.

Contingent estimate: new production `+96/-0`, existing `+16/-4`, tests
`+128/-0`, total `+240/-4`, **NET 236**, **CHURN 244**. It is reduced from
+258/-4 by removing forbidden final-accept/epoch work and tests. It is below
300; if reforecast exceeds 300, stop for a security-cohesive split. Never
combine B1c1a+b as a candidate.

Future tests own seven-field authorized success with metadata/capabilities but
no wire accept/epoch; safe identity/center reject; deny close/no JSON; exception
and unusable-hello failure; version/capability/timeout behavior; and absence of
legacy bypass, ACTIVE, and product eligibility. This docs-only slice adds no
runtime tests.
