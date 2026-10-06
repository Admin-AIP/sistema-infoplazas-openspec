# Gateway Application Handoff + Transport Validation

## Status: NOT STARTED — dependent on B1c1a1 COMPLETE + INTEGRATED

## Decision

B1c1a2 delivers the minimal safe post-mTLS/WSS application handoff needed for
later B1c1b composition. It starts only after B1c1a1 is **COMPLETE +
INTEGRATED**, consumes that integrated codec, and **must not duplicate**
parsing or validation.

## Planned surfaces and delivery gate

| Surface | Planned role |
| --- | --- |
| `apps/desktop/electron/security/transport/tls-gateway.ts` | minimal application-handoff seam |
| `apps/desktop/test/main/security/transport/tls-gateway.test.ts` | handoff, retention, failure, and cleanup tests |

Forecast: gateway **+18/-1** and tests **+91/-1**, total **+109/-2**, **NET
107**, **CHURN 111**. This passes the <=300 planning checkpoint. Run a fresh
exact checkpoint immediately before implementation; if it exceeds 300 NET,
stop and propose a cohesive split.

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
