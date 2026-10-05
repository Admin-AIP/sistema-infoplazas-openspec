# PR-08B1b2b2 Claim Binding + Final Authorization Design

## Status: NOT STARTED

This document authorizes architecture and planning only. PR-08B1b2b2 implementation has not started.

## Decision

PR-08B1b2b2 validates untrusted peer claims against deny-cleared, registry-derived authority and produces the final `AuthorizedPeerIdentity`. Its public API owns the complete deny-before-claims boundary:

```text
RegistryResolvedInstallation
→ evaluateDeny()
→ DenyClearedAuthorization
→ exact claim binding
→ AuthorizedPeerIdentity
```

No normal public path may produce `AuthorizedPeerIdentity` without first passing `evaluateDeny()`. This slice adds no registry lookup, deny logic, transport behavior, FSM composition, `ACTIVE` transition, `link.accept`, durable audit, business handler, or product action.

## Dependency and Baseline

**Dependency**: PR-08B1b2b1 — Contract Bridge + Deny Evaluation

**Dependency status**: COMPLETE + INTEGRATED

**Integrated Dinamizador commit**: `97164adf0b0d27ccbce04fbd019d3afa8b0fbe2e`

PR-08B1b2b2 consumes the integrated B1b2b1 contracts without modifying them:

- `RegistryResolvedInstallation`
- `DenyRegistry`
- `DenyClearedAuthorization`
- `DenyEvaluationError`
- `DenyEvaluationResult`
- `evaluateDeny()`

No B1b2b1 change is required.

## Scope

### Included

1. Reuse the existing `UntrustedPeerClaims` contract from `identity.ts`.
2. Delegate deny evaluation to B1b2b1 before reading or comparing claims.
3. Bind station, source, and center claims using exact equality.
4. Produce the seven-field final `AuthorizedPeerIdentity` explicitly from deny-cleared registry authority.
5. Return the existing deny errors and claim-binding errors through `SecureLinkResult`.
6. Preserve the legacy `promoteStationIdentity()` implementation without using it in the new PR-08 path.

### Excluded

- New registry lookup behavior
- SAN parsing
- Certificate extraction
- TLS or transport changes
- Credential allowlist changes
- New deny evaluation logic
- Legacy identity-promotion refactoring
- FSM composition
- `ACTIVE` transition
- `link.accept` emission
- Clock-plausibility resolution
- Durable `SecurityAudit`
- Business handlers
- Product actions: **ZERO**

## Reconciled Contracts

### Existing UntrustedPeerClaims

B1b2b2 MUST import and reuse the existing TypeScript type from `identity.ts`:

```typescript
Readonly<{
  kind: "untrusted-peer-claims"
  stationId: string
  centerId: string
  sourceId: string
}>
```

B1b2b2 MUST NOT declare a second claims type. Earlier `station_id`, `center_id`, and `source_id` wording described conceptual or wire fields; it did not authorize another TypeScript contract. Reusing the existing camelCase, discriminated type is contract reconciliation, not an architecture change.

### AuthorizedPeerIdentity

The final identity contains exactly:

```typescript
Readonly<{
  installationId: InstallationId
  stationId: StationId | undefined
  centerId: CenterId
  sourceId: SourceId
  endpointRole: "usuario-pc" | "dinamizador"
  credentialFingerprint256: string
  registryVersion: number
}>
```

All seven values come from the authoritative `DenyClearedAuthorization`. Construction MUST select these fields explicitly.

The implementation MUST NOT spread the complete intermediate object. `serialNumber` remains credential evidence and MUST NOT appear in `AuthorizedPeerIdentity`.

Claims are comparison inputs only. They never populate, overwrite, repair, canonicalize, or replace final identity values.

### Errors and Result

```typescript
ClaimAuthorizationError =
  | "IDENTITY_MISMATCH"
  | "CENTER_MISMATCH"

AuthorizationError =
  | DenyEvaluationError
  | ClaimAuthorizationError

AuthorizationResult = SecureLinkResult<
  AuthorizedPeerIdentity,
  AuthorizationError
>
```

`DenyEvaluationError` contributes:

- `INSTALLATION_DENIED`
- `CREDENTIAL_DENIED`

B1b2b2 reuses `SecureLinkResult`; it MUST NOT introduce a parallel result abstraction.

### Public API

```typescript
authorizePeer(
  registryResolved: RegistryResolvedInstallation,
  claims: UntrustedPeerClaims,
  denyRegistry: DenyRegistry
): Promise<AuthorizationResult>
```

The claim-binding helper, if extracted, MUST be private and non-exported. Tests MUST exercise behavior through `authorizePeer()` rather than exposing a deny-bypassing helper for convenience.

## Mandatory Authorization Sequence

`authorizePeer()` MUST execute in this order:

1. Call B1b2b1 `evaluateDeny(registryResolved, denyRegistry)` first.
2. Return an installation or credential deny error immediately.
3. Perform claim binding only after deny success produces `DenyClearedAuthorization`.
4. Produce `AuthorizedPeerIdentity` only after every required exact comparison succeeds.

Caller discipline is insufficient. The exported API itself enforces this sequence.

## Exact Claim-Binding Rules

### stationId: Presence-Based

When authoritative `stationId !== undefined`:

```text
claims.stationId === authoritative stationId
```

A mismatch returns `IDENTITY_MISMATCH`.

When authoritative `stationId === undefined`, the station claim is not compared. This rule depends on authoritative station presence, not solely on `endpointRole`.

No station claim may be normalized, derived, repaired, or used to populate a missing registry station.

### sourceId: Mandatory for Every Role

For every peer role, independently of station presence:

```text
claims.sourceId === authoritative sourceId
```

A mismatch returns `IDENTITY_MISMATCH`.

This comparison remains mandatory when authoritative `stationId` is undefined. There is no trim, case folding, Unicode normalization, fallback, repair, or derivation from `stationId`.

These two statements are intentionally distinct:

- **sourceId semantics**: OPEN
- **sourceId authorization binding**: EXACT and mandatory for all roles

Exact opaque-value binding does not resolve the semantic relationship between `sourceId` and `stationId`.

### centerId: Mandatory for Every Role

For every peer role:

```text
claims.centerId === authoritative centerId
```

A mismatch returns `CENTER_MISMATCH`.

There is no trim, case folding, Unicode normalization, fallback, repair, or claim-derived replacement.

## Legacy promoteStationIdentity Boundary

The existing exported `promoteStationIdentity()` remains unchanged:

- It currently has no production callers.
- Its current callers are tests.
- It is a legacy identity-promotion primitive.
- It does not perform B1b2a registry/credential authorization.
- It does not perform B1b2b1 deny evaluation.
- It produces `TrustedStationIdentity`, not `AuthorizedPeerIdentity`.

Treatment: **KEEP as legacy-only existing code**.

B1b2b2 MUST NOT reuse, adapt, delete, or refactor it. The implementation MUST NOT fabricate `AuthenticatedPeerContext` or `AuthorizedStationRecord` values to reuse the legacy function.

For the new PR-08 secure-link path, `promoteStationIdentity()` and `TrustedStationIdentity` are not alternatives to `authorizePeer()` and `AuthorizedPeerIdentity`. Future B1c production composition MUST consume the successful B1b2b2 authorization result rather than bypassing it through legacy promotion.

## Authority Boundaries

| Value | Authority |
| --- | --- |
| `installationId` | Certificate-derived chain through registry resolution |
| `credentialFingerprint256` | Certificate-derived evidence through registry resolution |
| `serialNumber` | Certificate-derived deny evidence only; excluded from final identity |
| `endpointRole` | Registry-derived |
| `stationId` | Registry-derived; may be undefined |
| `centerId` | Registry-derived |
| `sourceId` | Registry-derived opaque value |
| `registryVersion` | Registry-derived |
| Network claims | Non-authoritative comparison inputs only |

## Clock Plausibility

Clock plausibility remains **OPEN**.

It does not block B1b2b2 claim authorization or production of `AuthorizedPeerIdentity`.

It continues to block `ACTIVE` and production acceptance later in B1c2. B1b2b2 MUST NOT solve, infer, or bypass clock plausibility.

## Required Verification Coverage

Tests may be table-driven when that improves clarity and removes repeated setup. Equivalent explicit evidence may cover multiple bullets; meaningful security scenarios MUST NOT be removed to meet the line budget.

### Claims

- Station exact equality when authoritative station is present
- Station mismatch → `IDENTITY_MISMATCH`
- Station claim skipped when authoritative station is undefined
- Source exact equality for a stationful peer
- Source mismatch for a stationful peer → `IDENTITY_MISMATCH`
- Source exact equality for a stationless peer
- Source mismatch for a stationless peer → `IDENTITY_MISMATCH`
- Center exact equality
- Center mismatch → `CENTER_MISMATCH`
- No trim
- No case folding
- No Unicode or other normalization

### Final Identity

- Successful `AuthorizedPeerIdentity`
- Exact seven-field output
- `serialNumber` absent
- Registry values preserved
- Claims cannot overwrite or repair registry values
- Stationless final identity preserves `stationId: undefined`

### Deny Before Claims

- Installation deny prevents claim binding
- Fingerprint deny prevents claim binding
- Serial deny prevents claim binding
- Deny-registry exception prevents claim binding
- Successful deny evaluation is required before final authorization

### Boundaries

- Claim helper remains private/non-exported
- No legacy `promoteStationIdentity()` bypass to `AuthorizedPeerIdentity`
- No `ACTIVE`
- No `link.accept`
- No FSM composition
- `sourceId` semantics remain OPEN
- Product actions remain ZERO

Boundary exclusions should be verified structurally through the changed surface and import graph rather than through comment-only tests.

## Size Forecast and Planning Ceiling

Reconciled full Git forecast:

| Category | Insertions | Deletions | NET | CHURN |
| --- | ---: | ---: | ---: | ---: |
| Production | 74 | 0 | 74 | 74 |
| Tests | 224 | 0 | 224 | 224 |
| Existing-file modifications | 0 | 0 | 0 | 0 |
| **Total** | **298** | **0** | **298** | **298** |

Planning decision:

- `<=300` pre-implementation checkpoint: **PASS**
- Planning headroom: **2 NET lines**
- Pre-split required now: **NO**
- Actual final integration gate: `<=400` NET

The 298-NET forecast is the implementation planning ceiling. The `<=400` final gate does not authorize uncontrolled growth.

Before or during implementation, if scope or test-design changes make the projected candidate exceed 300 NET, work MUST stop before continuing and return for a security-cohesive split decision. Meaningful security coverage MUST NOT be deleted, weakened, hidden, or compressed merely to stay under 300.

## Security Invariants

- `authorizePeer()` is the sole normal public producer of `AuthorizedPeerIdentity`.
- Deny evaluation always precedes claim binding.
- Deny and deny-registry failures return before claims are evaluated.
- Claim matching is exact and fail-closed.
- Source binding is mandatory for stationful and stationless peers.
- Registry/certificate-derived values remain authoritative.
- Claims remain non-authoritative.
- Final identity construction cannot leak `serialNumber`.
- Legacy promotion cannot bypass the new PR-08 authorization path.
- Clock plausibility remains a later `ACTIVE`/production gate.
- Product actions remain ZERO.

## Next Slice

### B1c2 — ACTIVE / Production Acceptance

B1c2 may consume the successful B1b2b2 authorization result only after its own prerequisites, including clock plausibility, are satisfied. It remains outside PR-08B1b2b2.
