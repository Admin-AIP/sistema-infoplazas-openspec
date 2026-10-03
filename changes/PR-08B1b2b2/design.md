# PR-08B1b2b2 Claim Binding + Final Authorization Design

## Decision

PR-08B1b2b2 validates untrusted network claims against registry-derived authoritative identity and produces the final `AuthorizedPeerIdentity`. It consumes deny-cleared intermediate results from B1b2b1, performs exact claim matching with fail-closed semantics, and produces FINAL authorization. The public API enforces deny-before-claims order. It performs no registry lookup, no deny evaluation, no ACTIVE transition, and no product action.

This is the second subdivision of PR-08B1b2b, mandated by the >450 NET size threshold.

## Subdivision Context

**Parent work**: PR-08B1b2b (Deny + Authorization Composition)  
**Split reason**: Mandatory size split (456 NET > 450 threshold)  
**Previous slice**: B1b2b1 — Contract Bridge + Deny Evaluation (forecast: ~210 NET)  
**This slice**: B1b2b2 — Claim Binding + Final Authorization (forecast: ~189 NET)

**Dependency**: PR-08B1b2b1 MUST be integrated before B1b2b2 implementation begins.

## Scope

### Included

1. **UntrustedPeerClaims type** (network claims from link.hello):
   - `station_id`, `center_id`, `source_id`
   - NON-AUTHORITATIVE semantics

2. **Exact claim matching**:
   - Station/source mismatch → IDENTITY_MISMATCH
   - Center mismatch → CENTER_MISMATCH
   - Exact string equality (no normalization, trim, case-fold)
   - Fail-closed on mismatch

3. **Final AuthorizedPeerIdentity production**:
   - installationId, stationId, centerId, sourceId, endpointRole
   - credentialFingerprint256, registryVersion
   - Does NOT include serialNumber (credential evidence only)

4. **Public end-to-end composition**: `authorizePeer()`
   - Input: RegistryResolvedInstallation + UntrustedPeerClaims + DenyRegistry
   - Calls B1b2b1 `evaluateDeny()` FIRST
   - Only after deny success: claim matching
   - Output: AuthorizedPeerIdentity
   - Enforces deny-before-claims order at API boundary

5. **Fail-closed behavior**:
   - Station mismatch → IDENTITY_MISMATCH
   - Source mismatch → IDENTITY_MISMATCH
   - Center mismatch → CENTER_MISMATCH
   - Any claim mismatch rejects

### Excluded

- Registry lookup (B1b2a scope)
- Deny evaluation logic (B1b2b1 scope)
- SAN parsing (B1b1a scope)
- Certificate extraction (B1b1b scope)
- ACTIVE transition (B1c scope)
- link.accept emission (B1c scope)
- FSM composition (B1c scope)
- Durable audit (B2 scope)
- Product actions: **ZERO**

## Public Contract

```typescript
// Untrusted network claims (NON-AUTHORITATIVE)
export type UntrustedPeerClaims = Readonly<{
  station_id: string
  center_id: string
  source_id: string
}>

// Final authorized peer identity (FINAL authorization)
export type AuthorizedPeerIdentity = Readonly<{
  installationId: InstallationId
  stationId: StationId | undefined
  centerId: CenterId
  sourceId: SourceId
  endpointRole: 'usuario-pc' | 'dinamizador'
  credentialFingerprint256: string
  registryVersion: number
}>

// Claim authorization errors
export type ClaimAuthorizationError =
  | 'IDENTITY_MISMATCH'
  | 'CENTER_MISMATCH'

// Combined authorization error (deny + claim)
export type AuthorizationError =
  | DenyEvaluationError  // from B1b2b1
  | ClaimAuthorizationError

export type AuthorizationResult = SecureLinkResult<
  AuthorizedPeerIdentity,
  AuthorizationError
>

// Public end-to-end authorization (deny-before-claims enforced)
export async function authorizePeer(
  registryResolved: RegistryResolvedInstallation,
  claims: UntrustedPeerClaims,
  denyRegistry: DenyRegistry
): Promise<AuthorizationResult>
```

The public `authorizePeer()` function enforces deny-before-claims order and prevents bypass through the public API.

## Implementation Strategy

### Step 1: Deny Evaluation (via B1b2b1)

```typescript
export async function authorizePeer(
  registryResolved: RegistryResolvedInstallation,
  claims: UntrustedPeerClaims,
  denyRegistry: DenyRegistry
): Promise<AuthorizationResult> {
  // Call B1b2b1 evaluateDeny() FIRST
  const denyResult = await evaluateDeny(registryResolved, denyRegistry)
  
  if (!denyResult.ok) {
    // Deny evaluation failed → stop, return deny error
    return denyResult
  }

  // Deny evaluation succeeded → continue to claim matching
  const denyClearedAuth = denyResult.value
  
  // Step 2: Claim matching (internal helper or inline)
  return bindClaimsToIdentity(denyClearedAuth, claims)
}
```

### Step 2: Station/Source Claim Matching

```typescript
// Station/source claim matching (usuario-pc only)
if (denyClearedAuth.stationId !== undefined) {
  if (claims.station_id !== denyClearedAuth.stationId ||
      claims.source_id !== denyClearedAuth.sourceId) {
    return { ok: false, error: 'IDENTITY_MISMATCH' }
  }
}
```

**Exact equality**: No trim, no case-fold, no normalization.

### Step 3: Center Claim Matching

```typescript
// Center claim matching (all roles)
if (claims.center_id !== denyClearedAuth.centerId) {
  return { ok: false, error: 'CENTER_MISMATCH' }
}
```

### Step 4: Produce Final AuthorizedPeerIdentity

```typescript
return {
  ok: true,
  value: {
    installationId: denyClearedAuth.installationId,
    stationId: denyClearedAuth.stationId,
    centerId: denyClearedAuth.centerId,
    sourceId: denyClearedAuth.sourceId,
    endpointRole: denyClearedAuth.endpointRole,
    credentialFingerprint256: denyClearedAuth.credentialFingerprint256,
    registryVersion: denyClearedAuth.registryVersion,
  },
}
```

**Note**: `serialNumber` is NOT included in final AuthorizedPeerIdentity (credential evidence only).

## Deny-Before-Claims Enforcement

### Public API contract

The public `authorizePeer()` function MUST:
1. Call `evaluateDeny()` from B1b2b1
2. STOP on deny error
3. Only after deny success: perform claim matching
4. Return AuthorizedPeerIdentity

### Internal helper (optional)

If an internal `bindClaimsToIdentity()` helper is used for claim matching:

```typescript
function bindClaimsToIdentity(
  denyClearedAuth: DenyClearedAuthorization,
  claims: UntrustedPeerClaims
): SecureLinkResult<AuthorizedPeerIdentity, ClaimAuthorizationError>
```

**Keep it NON-PUBLIC** unless there is a concrete architecture reason to expose it.

### No bypass path

There MUST be no normal public API that produces `AuthorizedPeerIdentity` without deny evaluation first. The public `authorizePeer()` enforces this at the API boundary.

## Claim Authority Boundary

**Network claims are NON-AUTHORITATIVE**:
- Claims are compared against registry-derived authoritative values
- Claims NEVER override, select, or repair registry identity
- Mismatch → fail-closed rejection
- Claims prove knowledge but do not grant authority

**Registry-derived values are authoritative**:
- `stationId`, `centerId`, `sourceId` come from registry (via B1b2a)
- Claims must match registry values exactly
- Final AuthorizedPeerIdentity carries registry values, not claim values

## Test Strategy

### Minimum coverage (17 tests)

**Claim matching**:
1. station_id mismatch → IDENTITY_MISMATCH
2. source_id mismatch → IDENTITY_MISMATCH
3. center_id mismatch → CENTER_MISMATCH
4. exact station equality (no normalization)
5. exact source equality (no trim/case-fold)
6. exact center equality (no normalization)

**Successful authorization**:
7. all checks pass → AuthorizedPeerIdentity
8. final identity has exact fields
9. serialNumber NOT in final AuthorizedPeerIdentity
10. registry values preserved, claims never overwrite

**Deny-before-claims order**:
11. installation deny prevents claim evaluation
12. fingerprint deny prevents claim evaluation
13. serial deny prevents claim evaluation
14. credential deny checked before claim matching

**Role-specific**:
15. dinamizador role with undefined stationId succeeds

**Boundary**:
16. no ACTIVE transition
17. ZERO product actions

Use in-memory deny registry + claim fixtures inline with tests.

## Size Forecast

Based on oversized 456-NET implementation evidence:

- **Production**: 66 NET
  - claim-authorization.ts: +66/0 (UntrustedPeerClaims + AuthorizedPeerIdentity + authorizePeer + types)

- **Tests**: 123 NET
  - claim-authorization.test.ts: +123/0 (17 tests)

- **Total**: **189 NET**, 189 CHURN

**Gate check**:
- <=300 checkpoint: ✓ **PASS** (189 < 300)
- <=400 gate: ✓ **PASS** (189 < 400)

## Security

**Deny-before-claims invariant**:
- Public `authorizePeer()` calls `evaluateDeny()` FIRST
- Deny failure → stop, no claim evaluation
- Only deny success → claim matching
- No public bypass path

**Exact claim matching**:
- Exact string equality (no normalization)
- Registry values authoritative
- Claims prove knowledge, not authority
- Mismatch → fail-closed rejection

**Final authorization**:
- AuthorizedPeerIdentity is FINAL
- Produced only after deny + claim checks pass
- serialNumber NOT included (credential evidence only)
- No further authorization gates before B1c ACTIVE

**Authority boundaries preserved**:
- Certificate-derived: installationId, fingerprint256 (serialNumber excluded from final)
- Registry-derived: endpointRole, stationId, centerId, sourceId, registryVersion
- Network claims: NON-AUTHORITATIVE (comparison only)

**SourceId semantics**: OPEN (relationship to stationId unresolved)

## Product Actions

**ZERO**

This slice performs no registry lookup, no deny evaluation logic, no ACTIVE transition, no link.accept, and no product logic.

## Next Slice

**PR-08B1c**: ACTIVE Transition + link.accept Emission
- Consumes AuthorizedPeerIdentity
- Performs ACTIVE transition
- Emits link.accept
- Outside current PR-08B1b scope
