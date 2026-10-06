# PR-08B1c Secure-Link Delivery Reconciliation

## Status: NOT STARTED — design approved; implementation not authorized

## Decision

B1c1a is retained solely as an architectural umbrella. It is **not** an
implementation candidate and becomes complete only when both
[B1c1a1](../PR-08B1c1a1/design.md) and [B1c1a2](../PR-08B1c1a2/design.md) are
**COMPLETE + INTEGRATED**. The former 218-NET B1c1a forecast was superseded by
deeper read-only planning; no B1c1a implementation has started.

```text
B1b2b2 COMPLETE + INTEGRATED
  -> B1c1a1 PR-07 Envelope Codec + Focused Validation
  -> B1c1a2 Gateway Application Handoff + Transport Validation
  -> B1c1b Authorization + Negotiation Composition
  -> Clock Plausibility Decision
  -> B1c2 descendants
```

This documentation-only reconciliation authorizes no runtime work. B1b2b2
remains the completed authority: `authorizePeer()` is the sole normal producer
of the seven-field `AuthorizedPeerIdentity`.

## Review path

| Slice | State | Produces | Never produces |
| --- | --- | --- | --- |
| [B1c1a](../PR-08B1c1a/design.md) | ARCHITECTURAL UMBRELLA | sequencing and completion criteria | an implementation candidate |
| [B1c1a1](../PR-08B1c1a1/design.md) | NOT STARTED | decoded, untrusted PR-07 hello | gateway handoff, authority, accept, ACTIVE |
| [B1c1a2](../PR-08B1c1a2/design.md) | NOT STARTED; dependent on B1c1a1 integration | safe post-mTLS/WSS application handoff | parsing duplication, authority, accept, ACTIVE |
| [B1c1b](../PR-08B1c1b/design.md) | NOT STARTED; NOT READY | later authorized/negotiated, non-active internal result | `link.accept`, epoch, product eligibility |
| [B1c2](../PR-08B1c2/design.md) | BLOCKED / planning | later lifecycle children | an implementation candidate |

## Normative boundaries

The protocol authority is
[`design.md` §8.2–8.5](../session-start-login-usuario-pc/design.md#82-sobre-agent-v2-cerrado-para-pr-07)
and [`spec.md:255-445`](../session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md).
Wire fields remain snake_case even when TypeScript contracts are camelCase.

`link.hello` is an exact request with `source_id`, `station_id`, `center_id`,
and `{build_version, capabilities}`. The final correlated `link.accept`, a
fresh random nonzero `uint64` epoch, and the ACTIVE boundary belong only to a
future activation-capable B1c2 descendant. Product actions are **ZERO**
throughout every slice in this chain.

## Open gates

Clock Plausibility remains **OPEN**. It does not block B1c1a1, B1c1a2, or
B1c1b's internal non-active authorization/negotiation. It does block final
activation-capable accept and ACTIVE/production acceptance. `sourceId`
semantics remain **OPEN** and are not reopened by these slices. Durable audit,
production PKI, and protected server-cert provisioning remain later work.

## Forecast and delivery gates

The prior B1c1a estimate (NET 218) is superseded. Deeper read-only planning
forecast the unsplit monolith as follows:

| Area | Additions/deletions |
| --- | ---: |
| codec | +118/-0 |
| gateway | +18/-1 |
| codec tests | +145/-0 |
| gateway tests | +91/-1 |
| **Total** | **+372/-2** |

That is **NET 370**, **CHURN 374**: the mandatory <=300 planning checkpoint
**FAILS**. The <=400 final project gate does not override that failure. The
cohesive delivery split is B1c1a1 (+263/-0; NET 263; CHURN 263) followed by
B1c1a2 (+109/-2; NET 107; CHURN 111), never a combined candidate.

Each child requires a fresh exact checkpoint immediately before implementation.
If a child exceeds 300 NET, stop and propose another cohesive split; never
weaken validation. B1c1b remains NOT STARTED and NOT READY until both children
are COMPLETE + INTEGRATED, then must be reforecast from actual integrated
interfaces. Its old 236-NET estimate is not exact authority.
