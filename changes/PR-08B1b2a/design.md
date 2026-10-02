# PR-08B1b2a Installation Registry Resolution + Credential Authorization Design

## Decision

PR-08B1b2a implements registry-based installation lookup keyed ONLY by certificate-derived `installationId` and validates the presented certificate credential against the registry's active credential allowlist. It produces a non-final intermediate result that carries registry-authoritative identity and bindings. It performs no deny checks, no final claim binding, and no product action.

This is the first subdivision of the original PR-08B1b2 work, mandated by the >350 NET pre-implementation checkpoint.

## Subdivision Context

**Original monolithic scope**: PR-08B1b2 (forecast: 360-450 NET, exceeded checkpoint)  
**Split reason**: Mandatory size checkpoint enforcement at >350 NET projected  
**This slice**: B1b2a — Registry resolution + credential matching (forecast: 220-260 NET)  
**Next slice**: B1b2b — Deny + binding authorization (depends on B1b2a, forecast: 170-210 NET)

**Dependency**: PR-08B1b1b MUST be integrated before B1b2a implementation begins. B1b2a produces an intermediate result consumed by B1b2b.

## Scope

### Included

- `InstallationRegistry` interface definition
- Registry lookup keyed ONLY by `ParsedCertificateIdentity.installationId`
- Unknown installation → fail-closed rejection (`INSTALLATION_UNKNOWN`)
- Inactive/unauthorized registry state → fail-closed rejection (`INSTALLATION_UNAUTHORIZED`)
- Endpoint role validation (`usuario-pc` vs `dinamizador`) → `ROLE_MISMATCH` on mismatch
- Credential fingerprint256 matching against registry active allowlist
- Credential serial matching against registry active allowlist (when applicable)
- Registry version propagation
- Registry-derived `stationId` (opaque, Usuario PC only)
- Registry-derived `centerId` (opaque)
- Registry-derived `sourceId` (opaque, semantic relationship to `stationId` remains OPEN)
- Intermediate non-final `RegistryResolvedInstallation` result type

### Excluded

- Deny/revocation registry (B1b2b scope)
- Deny precedence logic (B1b2b scope)
- Final claim-binding composition (B1b2b scope)
- Final `AuthorizedPeerIdentity` type (B1b2b scope)
- ACTIVE transition (B1c scope)
- `link.accept` emission (B1c scope)
- B1c FSM composition
- Business handlers
- Product actions: **ZERO**

## Public Contract

```typescript
// Installation registry interface
export interface InstallationRegistry {
  lookup(installationId: InstallationId): Promise<InstallationRegistryRecord | undefined>
}

export type InstallationRegistryRecord = Readonly<{
  installationId: InstallationId
  endpointRole: 'usuario-pc' | 'dinamizador'
  stationId: StationId | undefined  // usuario-pc only
  centerId: CenterId
  sourceId: SourceId
  authorizationState: 'active' | 'inactive' | 'suspended'
  registryVersion: number
  allowedCredentials: readonly AllowedCredential[]
}>

export type AllowedCredential = Readonly<{
  fingerprint256?: string
  serialNumber?: string
  validFrom?: Date
  validUntil?: Date
}>

// Intermediate result (NOT final authorization)
export type RegistryResolvedInstallation = Readonly<{
  installationId: InstallationId
  endpointRole: 'usuario-pc' | 'dinamizador'
  stationId: StationId | undefined
  centerId: CenterId
  sourceId: SourceId
  credentialFingerprint256: string
  registryVersion: number
}>

export type RegistryResolutionError =
  | 'INSTALLATION_UNKNOWN'
  | 'INSTALLATION_UNAUTHORIZED'
  | 'ROLE_MISMATCH'
  | 'CREDENTIAL_NOT_ALLOWED'

export type RegistryResolutionResult = SecureLinkResult<
  RegistryResolvedInstallation,
  RegistryResolutionError
>

export function resolveInstallationRegistry(
  certificateIdentity: ParsedCertificateIdentity,
  expectedRole: 'usuario-pc' | 'dinamizador',
  registry: InstallationRegistry
): Promise<RegistryResolutionResult>
```

The function never throws. All registry exceptions are caught and mapped fail-closed.

## Implementation Strategy

### Step 1: Registry Lookup

```typescript
const record = await registry.lookup(certificateIdentity.installationId)

if (!record) {
  return { ok: false, error: 'INSTALLATION_UNKNOWN' }
}
```

Lookup uses ONLY the certificate-derived `installationId`. Network claims, CN/O/OU, IP, MAC, hostname, and display names NEVER select or repair registry identity.

### Step 2: Authorization State Check

```typescript
if (record.authorizationState !== 'active') {
  return { ok: false, error: 'INSTALLATION_UNAUTHORIZED' }
}
```

Only `'active'` state proceeds. `'inactive'` and `'suspended'` are denied.

### Step 3: Endpoint Role Validation

```typescript
if (record.endpointRole !== expectedRole) {
  return { ok: false, error: 'ROLE_MISMATCH' }
}
```

The registry-declared role must match the expected role for this TLS endpoint.

### Step 4: Credential Allowlist Matching

```typescript
const credentialMatch = record.allowedCredentials.some(
  (allowed) =>
    (allowed.fingerprint256 === certificateIdentity.certificateFingerprint256) ||
    (allowed.serialNumber === certificateIdentity.serialNumber)
)

if (!credentialMatch) {
  return { ok: false, error: 'CREDENTIAL_NOT_ALLOWED' }
}
```

**Credential matching rule** (from approved architecture):
- Registry maintains an active credential allowlist with `fingerprint256` OR `serialNumber` per entry
- Certificate credential matches if **either** its fingerprint256 OR its serialNumber appears in the allowlist
- This prevents unauthorized CA-signed certificates from inheriting authorization merely by copying `installationId`

### Step 5: Produce Intermediate Result

```typescript
return {
  ok: true,
  value: {
    installationId: record.installationId,
    endpointRole: record.endpointRole,
    stationId: record.stationId,
    centerId: record.centerId,
    sourceId: record.sourceId,
    credentialFingerprint256: certificateIdentity.certificateFingerprint256,
    registryVersion: record.registryVersion,
  },
}
```

This is NOT final authorization. Deny checks and claim matching remain in B1b2b.

## Certificate Authority Boundary

**Certificate-derived authority** (from B1b1b):
- `installationId`
- `certificateFingerprint256`
- `serialNumber`

**Registry-derived authority**:
- `endpointRole`
- `stationId`
- `centerId`
- `sourceId`
- Authorization state
- `registryVersion`
- Allowed credential evidence

**Network claims**: NON-AUTHORITATIVE
- Claims are NEVER used to select or repair registry identity
- Claims are compared against registry-derived identity in B1b2b, not here

## SourceId Semantics

**SourceId semantic relationship to stationId remains OPEN.**

This slice:
- Carries `sourceId` from the authoritative registry record
- Preserves it as an opaque registry-derived value
- Does NOT decide whether `sourceId` is independent or deterministically equal to `stationId`

The open requirement is preserved for future resolution.

## Security

**Fail-closed behavior**:
- Unknown installation → denied
- Inactive/unauthorized state → denied
- Role mismatch → denied
- Credential not in allowlist → denied
- Registry lookup exception → denied (caught and mapped)
- No fallback, no repair, no network-claim override

**Registry uniqueness**:
- `installationId` is a unique key
- Duplicate or inconsistent records → fail-closed (reject)

**No deny bypass**:
- This slice does NOT check deny/revocation
- B1b2b MUST apply deny checks before final authorization
- Registry resolution alone is insufficient for authorization

## Test Strategy

### Minimum test coverage

1. **Unknown installation**: `installationId` not in registry → `INSTALLATION_UNKNOWN`
2. **Inactive installation**: `authorizationState: 'inactive'` → `INSTALLATION_UNAUTHORIZED`
3. **Suspended installation**: `authorizationState: 'suspended'` → `INSTALLATION_UNAUTHORIZED`
4. **Role mismatch**: Expected `'usuario-pc'`, registry has `'dinamizador'` → `ROLE_MISMATCH`
5. **Credential not allowed**: Certificate fingerprint/serial not in allowlist → `CREDENTIAL_NOT_ALLOWED`
6. **Credential allowed by fingerprint**: Fingerprint match → success
7. **Credential allowed by serial**: Serial match → success
8. **Registry version propagation**: `registryVersion` copied to output
9. **StationId propagation**: Registry `stationId` copied (opaque)
10. **CenterId propagation**: Registry `centerId` copied (opaque)
11. **SourceId propagation**: Registry `sourceId` copied (opaque, semantic relationship OPEN)
12. **No network claim authority**: Claims not used for lookup (boundary test)
13. **No deny behavior**: This slice does not check deny (B1b2b owns that)

### Test doubles

Use in-memory test registry snapshots inline with tests. No separate fixture files needed.

## Size Forecast

- **Production**: 90-110 NET
  - Interface/types: ~25 lines
  - Lookup logic: ~40 lines
  - Role validation: ~10 lines
  - Credential matching: ~20 lines
  - Error mapping: ~15 lines
- **Tests**: 120-140 NET
  - Unknown installation: ~20 lines
  - Unauthorized states: ~20 lines
  - Role mismatch: ~15 lines
  - Credential scenarios: ~30 lines
  - Registry version: ~10 lines
  - Opaque field transport: ~15 lines
  - Boundary tests: ~15 lines
- **Modifications**: ~10 NET (identity.ts, contracts.ts for new types)
- **Total**: **220-260 NET**

Well within <=350 checkpoint and <=400 gate.

## Product Actions

**ZERO**

This slice performs no ACTIVE transition, no `link.accept`, no FSM composition, and no business handlers.
