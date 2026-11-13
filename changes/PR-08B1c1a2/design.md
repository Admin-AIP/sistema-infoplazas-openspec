# Gateway Application Handoff + Transport Validation

## Status: COMPLETE + INTEGRATED — remote `master` contains `2fad54c`

## Decision

B1c1a2 delivered the minimal safe post-mTLS/WSS application handoff needed for
later B1c1b composition. It consumed the integrated B1c1a1 codec without
duplicating parsing or validation. The initial implementation was `7f870d9`;
a bounded corrective follow-up is integrated as `2fad54c`.

## Delivered surfaces and evidence

| Surface | Delivered role |
| --- | --- |
| `apps/desktop/electron/security/transport/tls-gateway.ts` | minimal application-handoff seam with callback-time generation protection |
| `apps/desktop/test/main/security/transport/tls-gateway.test.ts` | handoff, pre-gate non-invocation, stale-generation, failure cleanup, and recovery tests |

The initial implementation checkpoint against `3e4f236` was **+141/-3, NET
138, CHURN 144**. The final reconciled corrective checkpoint against `73dc548`
was **+266/-8, NET 258, CHURN 274**, within the <=300 limit. The delivered correction passed 76 focused gateway
tests, 308 `main/security` tests, typecheck, the official emitted build, formal
QA, and independent architecture review. It introduced no additional runtime
scope beyond these two paths.

## Handoff boundary

The API is subject to design inspection and must remain narrow rather than
becoming a generic framework. The seam may retain only what the handoff needs:

- accepted `WebSocket` and request/upgrade context as necessary;
- authenticated live `TLSSocket`, or a narrow read-only credential context;
- decoded application frame/context from the integrated codec.

It must retain enough authenticated live context for the existing
`extractPeerCertificateIdentity()` path. It must not copy, serialize, or
fabricate a raw-certificate DTO, and it does not perform B1b authorization.

## Required transport gates and failure behavior

Handoff occurs only after every existing gateway gate has passed: TLS 1.3,
mTLS, TLS-layer authorization, certificate date validation, synchronous
resumed-session rejection, HTTPS/WSS upgrade validation, connection tracking,
heartbeat/resource hardening, and cleanup setup. The callback must not run for
unauthorized TLS, bad certificate dates, a resumed session, failed upgrade, or
failed codec validation.

Design and tests must cover retained context, callback failure, socket and
tracking cleanup, no stale context, and no second lifecycle. Every failure is
fail-closed. Existing TLS failures remain transport/local-audit behavior and
are never converted into a JSON reply.

## Exclusions and gates

B1c1a2 excludes codec redefinition; registry, deny, claims, or
`authorizePeer`; capability negotiation; `link.reject`/`link.accept`; epoch;
ACTIVE; Clock Plausibility; sequence/replay/lifecycle; durable audit
persistence; and product actions. Product actions are **ZERO**.

Clock Plausibility remains OPEN but does not block this non-active handoff.
`sourceId` semantics remain OPEN and are not reopened. This slice is complete
only after implementation is integrated. That integration, together with
B1c1a1's, is the prerequisite for B1c1b reforecast and readiness.
