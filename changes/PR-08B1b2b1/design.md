# PR-08B1b2b1 Contract Bridge + Deny Evaluation Design

## Decision

PR-08B1b2b1 extends the intermediate contract with certificate-derived serialNumber and implements absolute deny precedence evaluation. It consumes `RegistryResolvedInstallation` from B1b2a, evaluates installation and credential deny checks with fail-closed exception handling, and produces a distinct deny-cleared intermediate result. It performs no claim validation, no final authorization, no ACTIVE transition, and no product action.

This is the first subdivision of PR-08B1b2b, mandated by the >450 NET size threshold.

## Subdivision Context

**Parent work**: PR-08B1b2b (Deny + Authorization Composition)  
**Split reason**: Mandatory size split (456 NET > 450 threshold)  
**This slice**: B1b2b1 — Contract Bridge + Deny Evaluation (forecast: ~210 NET)  
**Next slice**: B1b2b2 — Claim Binding + Final Authorization (forecast: ~189 NET)

**Dependency**: PR-08B1b2a MUST be integrated before B1b2b1 implementation begins.

## Scope

### Included

1. **Contract bridge**: RegistryResolvedInstallation.serialNumber extension
   - Type: add `serialNumber: string` field
   - Propagation: `certificateIdentity.serialNumber`
   - Authority: certificate-derived credential evidence
   - Tests: serialNumber propagation + authority boundary

2. **DenyRegistry interface** with explicit namespaces:
   - `checkInstallationDeny(installationId)`
   - `checkFingerprintDeny(fingerprint256)`
   - `checkSerialDeny(serialNumber)`

3. **Deny evaluation logic**:
   - Step 1: Installation deny (FIRST, absolute precedence)
   - Step 2: Credential deny (fingerprint OR serial)
   - Deny exception handling (fail-closed)
   - ANY deny hit rejects

4. **Distinct deny-cleared intermediate type**: `DenyClearedAuthorization`
   - Structurally equivalent to RegistryResolvedInstallation
   - Semantically: registry-resolved AND deny-cleared
   - Prevents accidental bypass: only `evaluateDeny()` produces it
   - Branded/opaque representation preferred if repository conventions allow

5. **Fail-closed behavior**:
   - Installation denied → INSTALLATION_DENIED
   - Fingerprint denied → CREDENTIAL_DENIED
   - Serial denied → CREDENTIAL_DENIED
   - Deny lookup exception → fail-closed (deny error)

### Excluded

- Network claim comparison (B1b2b2 scope)
- Final AuthorizedPeerIdentity production (B1b2b2 scope)
- ACTIVE transition (B1c scope)
- link.accept emission (B1c scope)
- FSM composition (B1c scope)
- Durable audit (B2 scope)
- Product actions: **ZERO**

## Public Contract

```typescript
// Deny registry interface with explicit namespaces
export interface DenyRegistry {
  checkInstallationDeny(installationId: InstallationId): Promise<boolean>
  checkFingerprintDeny(fingerprint256: string): Promise<boolean>
  checkSerialDeny(serialNumber: string): Promise<boolean>
}

// Distinct deny-cleared intermediate (NON-FINAL)
// Produced ONLY by successful evaluateDeny()
export type DenyClearedAuthorization = Readonly<{
  installationId: InstallationId
  endpointRole: 'usuario-pc' | 'dinamizador'
  stationId: StationId | undefined
  centerId: CenterId
  sourceId: SourceId
  credentialFingerprint256: string
  serialNumber: string  // ← Contract bridge: certificate-derived evidence
  registryVersion: number
}>

// Deny evaluation errors
export type DenyEvaluationError =
  | 'INSTALLATION_DENIED'
  | 'CREDENTIAL_DENIED'

export type DenyEvaluationResult = SecureLinkResult<
  DenyClearedAuthorization,
  DenyEvaluationError
>

// Deny evaluation function
export async function evaluateDeny(
  registryResolved: RegistryResolvedInstallation,
  denyRegistry: DenyRegistry
): Promise<DenyEvaluationResult>
```

The function never throws. All deny exceptions are caught and mapped fail-closed.

## Implementation Strategy

### Step 1: Contract Bridge

Extend `RegistryResolvedInstallation` in installation-registry.ts:

```typescript
export type RegistryResolvedInstallation = Readonly<{
  // ... existing fields
  serialNumber: string  // ← NEW: certificate-derived credential evidence
  registryVersion: number
}>
```

Propagate in `resolveInstallationRegistry()`:

```typescript
return {
  ok: true,
  value: {
    // ... existing fields
    serialNumber: certificateIdentity.serialNumber,  // ← NEW
    registryVersion: record.registryVersion,
  },
}
```

### Step 2: Installation Deny (FIRST)

```typescript
let installationDenied: boolean
try {
  installationDenied = await denyRegistry.checkInstallationDeny(
    registryResolved.installationId
  )
} catch {
  return { ok: false, error: 'INSTALLATION_DENIED' }
}

if (installationDenied) {
  return { ok: false, error: 'INSTALLATION_DENIED' }
}
```

### Step 3: Credential Deny (SECOND)

```typescript
let fingerprintDenied: boolean
let serialDenied: boolean
try {
  fingerprintDenied = await denyRegistry.checkFingerprintDeny(
    registryResolved.credentialFingerprint256
  )
  serialDenied = await denyRegistry.checkSerialDeny(
    registryResolved.serialNumber
  )
} catch {
  return { ok: false, error: 'CREDENTIAL_DENIED' }
}

if (fingerprintDenied || serialDenied) {
  return { ok: false, error: 'CREDENTIAL_DENIED' }
}
```

### Step 4: Produce DenyClearedAuthorization

```typescript
return {
  ok: true,
  value: {
    installationId: registryResolved.installationId,
    endpointRole: registryResolved.endpointRole,
    stationId: registryResolved.stationId,
    centerId: registryResolved.centerId,
    sourceId: registryResolved.sourceId,
    credentialFingerprint256: registryResolved.credentialFingerprint256,
    serialNumber: registryResolved.serialNumber,
    registryVersion: registryResolved.registryVersion,
  },
}
```

## Deny Precedence

**Absolute precedence order**:
1. Installation deny (checked FIRST)
2. Credential deny: fingerprint OR serial (checked SECOND)
3. (Claim matching deferred to B1b2b2)

**Fail-closed**: ANY deny hit or exception rejects.

## Branded Type Recommendation

If repository conventions support it, prefer a branded representation:

```typescript
type DenyClearedBrand = { readonly __denyClearedBrand: unique symbol }
export type DenyClearedAuthorization = RegistryResolvedInstallation & DenyClearedBrand
```

This prevents direct construction:
```typescript
// ✗ Type error: missing brand
const fake: DenyClearedAuthorization = registryResolved

// ✓ Only evaluateDeny() can produce it
const result = await evaluateDeny(registryResolved, denyRegistry)
if (result.ok) {
  const cleared: DenyClearedAuthorization = result.value  // ✓ Valid
}
```

## Test Strategy

### Minimum coverage (13 tests)

**Contract bridge**:
1. serialNumber propagates from ParsedCertificateIdentity
2. certificateFingerprint256 propagates (existing)
3. registry lookup uses ONLY installationId, never serialNumber

**Installation deny**:
4. installation deny → INSTALLATION_DENIED

**Credential deny**:
5. fingerprint deny → CREDENTIAL_DENIED
6. serial deny → CREDENTIAL_DENIED
7. serial deny uses propagated serialNumber

**Deny precedence**:
8. installation deny checked before fingerprint deny
9. installation deny checked before serial deny

**Exception handling**:
10. installation deny exception → fail-closed INSTALLATION_DENIED
11. credential deny exception → fail-closed CREDENTIAL_DENIED

**Boundary**:
12. successful deny evaluation → DenyClearedAuthorization (NON-FINAL)
13. ZERO product actions

Use in-memory deny registry snapshots inline with tests.

## Size Forecast

Based on oversized 456-NET implementation evidence:

- **Production**: 89 NET
  - installation-registry.ts modifications: +4/-1 (contract bridge)
  - deny-evaluation.ts: +86/0 (DenyRegistry + evaluateDeny + types)

- **Tests**: 121 NET
  - installation-registry.test.ts: +27/0 (bridge tests)
  - deny-evaluation.test.ts: +94/0 (13 tests)

- **Total**: **210 NET**, 212 CHURN

**Gate check**:
- <=300 checkpoint: ✓ **PASS** (210 < 300)
- <=400 gate: ✓ **PASS** (210 < 400)

## Security

**Deny-cleared semantics**:
- DenyClearedAuthorization proves deny evaluation succeeded
- Only `evaluateDeny()` may produce it
- Branded type prevents accidental construction

**Fail-closed**:
- Installation denied → rejected
- Credential denied → rejected
- Deny exception → rejected
- No fallback, no bypass

**Authority boundaries preserved**:
- Certificate-derived: installationId, fingerprint256, serialNumber
- Registry-derived: endpointRole, stationId, centerId, sourceId, registryVersion
- Deny evaluation: installation + credential checks only

**SourceId semantics**: OPEN (relationship to stationId unresolved)

## Product Actions

**ZERO**

This slice performs no claim validation, no final authorization, no ACTIVE transition, no link.accept, and no product logic.

## Next Slice

**PR-08B1b2b2**: Claim Binding + Final Authorization
- Consumes DenyClearedAuthorization
- Validates network claims
- Produces final AuthorizedPeerIdentity
- Forecast: ~189 NET
