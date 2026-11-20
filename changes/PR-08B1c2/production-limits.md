# PR-08B1c2 Production limits

Status: ACCEPTED by the maintainer on 2026-11-20; documentation only. Authorizes no runtime implementation and no
`ACTIVE` behavior. Closes the `Production limits` row of `design.md` "Remaining blockers".

The WSS gateway of the Dinamizador (`apps/desktop/electron/security/transport/tls-gateway.ts`) requires
`maxConnections`, `maxPayload` and `pingIntervalMs` and has no defaults; the values used by its tests (10, 1 MiB,
5000 ms) are test values, not defaults. This document fixes the production defaults and who may change them.

## Evidence

* Typical Infoplaza: fewer than 5 Usuario PC stations in some centers, 6 to 9 in most, more than 10 in a few, almost
  never more than 15 (maintainer, 2026-11-20).
* Messages on the station link are short: requests, questions, information and answers. No measured message size
  exists yet; the first message (hello) carries only identifiers, claims, a build version and capabilities.
* The gateway sends a ping every `pingIntervalMs` and terminates a client that did not answer the previous ping at the
  next tick, so a dead peer is detected between one and two intervals later.

## Decision

| Parameter | Default | Permitted range | Reason |
| --- | --- | --- | --- |
| Authorized station quota (cupo) | 10 | 1 to 64 | Covers most centers without configuration. |
| `maxConnections` | 2 x the authorized station count, never below 8 (20 by default) | 8 to 128 | Room for a station that reconnects while its previous connection is still being closed. |
| `maxPayload` | 65 536 bytes (64 KiB) | 16 KiB to 1 MiB | Messages are short; the limit bounds the damage of an oversized frame. |
| `pingIntervalMs` | 15 000 ms | 5 000 to 60 000 ms | Detects a dead peer in 15 to 30 s without cutting a busy PC by mistake. |

## Who may change what

* More than 10 stations: the quota default stays 10. An authenticated Enlace or Operador del Centro may authorize
  additional stations, one by one, through the existing explicit approval of an over-quota enrollment of the
  secure-link specification; each approval raises the authorized station count of that center and `maxConnections`
  follows it. Every approval is audited.
* `maxPayload` and `pingIntervalMs` are technical settings: only the Enlace may change them, within the permitted range.
* A value outside its permitted range is rejected and the previous value stays in force; nothing is clamped silently.
* All changes are audited without payload and take effect when the gateway is restarted.

## Roles

Dinamizador is the program; Operador del Centro (Operador) is the person who operates the Infoplaza; the Enlace is the
human installer role with greater scope and privileges than the Operador (see the secure-link specification).

## Not decided here

The configuration mechanism and storage of these values; the exact audit event names; revisiting `maxPayload` once real
message types and sizes exist; limits of the cloud link; any Usuario PC conclusion; runtime implementation.
