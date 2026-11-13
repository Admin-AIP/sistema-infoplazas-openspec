# PR-08B1c1a Envelope Codec + Gateway Application Handoff Umbrella

## Status: ARCHITECTURAL UMBRELLA — COMPLETE + INTEGRATED

## Decision

B1c1a describes the cohesive delivery boundary formerly planned as one
candidate. It **MUST NOT** be implemented as one candidate. Its children are
complete and integrated in order:

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

| Child | Scope | Historical forecast | Delivered measurement and gate |
| --- | --- | ---: | --- |
| B1c1a1 | codec plus focused validation only; no gateway callback/handoff | +263/-0; NET 263; CHURN 263 | actual +260/-0; NET/CHURN 260 against `bcef13a`; COMPLETE + INTEGRATED at `3e4f236` |
| B1c1a2 | integrated-codec consumer plus gateway handoff/transport validation | +109/-2; NET 107; CHURN 111 | initial +141/-3; NET 138 against `3e4f236`; final +266/-8; NET 258, CHURN 274 against `73dc548`; COMPLETE + INTEGRATED at `2fad54c` |

Each delivered child checkpoint remains <=300 NET. No new B1c1a runtime work
is authorized by this umbrella.

## Shared gates and exclusions

Clock Plausibility remains **OPEN**. It does not block B1c1a1 or B1c1a2, but
it blocks final activation-capable accept and ACTIVE/production acceptance.
`sourceId` semantics remain OPEN and are not reopened. Within B1c1a, claims
remain untrusted; the separately integrated B1c1b boundary owns later
authorization and negotiation. Product actions remain **ZERO**.

B1c1a does not redefine protocol authority. See [B1c](../PR-08B1c/design.md)
for the delivery chain and [B1c1b](../PR-08B1c1b/design.md) for the separately
integrated, non-active authorization/negotiation composition.
