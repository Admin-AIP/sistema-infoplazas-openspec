# PR-08B1c1a Envelope Codec + Gateway Application Handoff Umbrella

## Status: ARCHITECTURAL UMBRELLA — NOT an implementation candidate

## Decision

B1c1a is retained only to describe the cohesive delivery boundary formerly
planned as one candidate. It **MUST NOT** be implemented as one candidate. It
is complete only when both children are **COMPLETE + INTEGRATED** in order:

```text
B1b2b2
  -> B1c1a1 PR-07 Envelope Codec + Focused Validation
  -> B1c1a2 Gateway Application Handoff + Transport Validation
  -> B1c1b
```

[B1c1a1](../PR-08B1c1a1/design.md) owns PR-07 outer-envelope decoding and
focused validation. [B1c1a2](../PR-08B1c1a2/design.md) consumes that integrated
codec and owns the minimal safe post-mTLS/WSS application handoff. Neither
child authorizes, negotiates capabilities, emits a final accept, creates an
epoch, transitions ACTIVE, or performs a product action.

## Why it was split

The prior forecast of +224/-6 (NET 218; CHURN 230) was superseded by deeper
read-only planning. The actual monolithic shape is forecast as:

| Area | Additions/deletions |
| --- | ---: |
| codec | +118/-0 |
| gateway | +18/-1 |
| codec tests | +145/-0 |
| gateway tests | +91/-1 |
| **Total** | **+372/-2** |

The result is **NET 370**, **CHURN 374**. It fails the mandatory <=300 planning
checkpoint. The <=400 final project gate cannot override that failure. No
implementation started under either estimate.

## Child delivery contracts

| Child | Scope | Forecast | Required gate |
| --- | --- | ---: | --- |
| B1c1a1 | codec plus focused validation only; no gateway callback/handoff | +263/-0; NET 263; CHURN 263 | fresh exact <=300 checkpoint before implementation |
| B1c1a2 | integrated-codec consumer plus gateway handoff/transport validation | +109/-2; NET 107; CHURN 111 | fresh exact <=300 checkpoint immediately before implementation |

If either checkpoint exceeds 300 NET, stop and propose a further cohesive
split. Validation must not be weakened to fit the gate.

## Shared gates and exclusions

Clock Plausibility remains **OPEN**. It does not block B1c1a1 or B1c1a2, but
it blocks final activation-capable accept and ACTIVE/production acceptance.
`sourceId` semantics remain OPEN and are not reopened. Claims stay untrusted
until the later B1c1b authorization boundary. Product actions remain **ZERO**.

B1c1a does not redefine protocol authority. See [B1c](../PR-08B1c/design.md)
for the delivery chain and [B1c1b](../PR-08B1c1b/design.md) for the dependent,
not-ready authorization/negotiation composition.
