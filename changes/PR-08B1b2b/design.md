# PR-08B1b2b Deny Registry + Authorization Composition Design

## Decision

PR-08B1b2b implements deny/revocation checks and final binding authorization composition. It consumes the intermediate `RegistryResolvedInstallation` from B1b2a, applies absolute deny precedence, validates network claims against registry-authoritative identity, and produces the final `AuthorizedPeerIdentity`. It performs no registry lookup, no ACTIVE transition, and no product action.

This is the second subdivision of the original PR-08B1b2 work, mandated by the >350 NET pre-implementation checkpoint.

## Subdivision Context

**Original monolithic scope**: PR-08B1b2 (forecast: 360-450 NET, exceeded checkpoint)  
**Split reason**: Mandatory size checkpoint enforcement at >350 NET projected  
**Previous slice**: B1b2a — Registry resolution + credential matching (220-260 NET)  
**This slice**: B1b2b — Deny + binding authorization (forecast: 170-210 NET)

**Dependency**: PR-08B1b2a MUST be integrated before B1b2b implementation begins.

## Scope

### Included

- `DenyRegistry` interface definition
- Deny/revocation lookup by `installationId`
- Deny/revocation lookup by credential `fingerprint256`
- Deny/revocation lookup by credential `serialNumber`
- **Absolute deny precedence**: DENY overrides active authorization, credential match, cutover, and matching claims
- Exact claim matching where required by architecture
  - `station_id` mismatch → `IDENTITY_MISMATCH`
  - `source_id` mismatch → `IDENTITY_MISMATCH`
  - `center_id` mismatch → `CENTER_MISMATCH`
- Final station/center/source binding
- Final `AuthorizedPeerIdentity` production
- Fail-closed composition

### Excluded

- SAN parsing (B1b1a scope)
- Certificate extraction (B1b1b scope)
- Registry lookup (B1b2a scope)
- TLS gateway implementation changes
- ACTIVE transition (B1c scope)
- `link.accept` emission (B1c scope)
- B1c FSM composition
- Durable SecurityAudit persistence (B2 scope)
- Business handlers
- Product actions: **ZERO**

## Public Contract

```typescript
// Deny registry interface
export interface DenyRegistry {
  checkDeny(installationId: InstallationId): Promise<boolean>
  checkCredentialDeny(fingerprint256: string): Promise<boolean>
  checkCredentialDeny(serialNumber: string): Promise<boolean>
}

// Untrusted network claims (from link.hello)
export type UntrustedPeerClaims = Readonly<{
  station_id: string
  center_id: string
  source_id: string
}>

// Final authorization result
export type AuthorizedPeerIdentity = Readonly<{
  installationId: InstallationId
  stationId: StationId | undefined  // usuario-pc only
  centerId: CenterId
  sourceId: SourceId
  endpointRole: 'usuario-pc' | 'dinamizador'
  credentialFingerprint256: string
  registryVersion: number
}>

export type AuthorizationError =
  | 'INSTALLATION_DENIED'
  | 'CREDENTIAL_DENIED'
  | 'IDENTITY_MISMATCH'  // station_id or source_id mismatch
  | 'CENTER_MISMATCH'    // center_id mismatch

export type AuthorizationResult = SecureLinkResult<
  AuthorizedPeerIdentity,
  AuthorizationError
>

export function authorizePeer(
  registryResolved: RegistryResolvedInstallation,
  claims: UntrustedPeerClaims,
  denyRegistry: DenyRegistry
): Promise<AuthorizationResult>
```

The function never throws. All deny/claim exceptions are caught and mapped fail-closed.

## Implementation Strategy

### Step 1: Deny by Installation

```typescript
const installationDenied = await denyRegistry.checkDeny(
  registryResolved.installationId
)

if (installationDenied) {
  return { ok: false, error: 'INSTALLATION_DENIED' }
}
```

**Deny precedence**: Installation deny is checked FIRST, before credential deny and before claim matching.

### Step 2: Deny by Credential

```typescript
const credentialDenied =
  (await denyRegistry.checkCredentialDeny(registryResolved.credentialFingerprint256)) ||
  (registryResolved.serialNumber && 
   await denyRegistry.checkCredentialDeny(registryResolved.serialNumber))

if (credentialDenied) {
  return { ok: false, error: 'CREDENTIAL_DENIED' }
}
```

**Deny keys** (from approved architecture):
- Deny may match by `installationId`, `fingerprint256`, or `serialNumber`
- **ANY deny hit rejects** (no fallback)

### Step 3: Exact Claim Matching

```typescript
// Station/source claim matching
if (registryResolved.stationId !== undefined) {
  if (claims.station_id !== registryResolved.stationId ||
      claims.source_id !== registryResolved.sourceId) {
    return { ok: false, error: 'IDENTITY_MISMATCH' }
  }
}

// Center claim matching
if (claims.center_id !== registryResolved.centerId) {
  return { ok: false, error: 'CENTER_MISMATCH' }
}
```

**Claim authority boundary**:
- Claims are NON-AUTHORITATIVE
- Claims are compared against registry-derived authoritative identity
- Claims NEVER override, select, or repair registry values
- Mismatch → fail-closed rejection

### Step 4: Produce Final Authorization

```typescript
return {
  ok: true,
  value: {
    installationId: registryResolved.installationId,
    stationId: registryResolved.stationId,
    centerId: registryResolved.centerId,
    sourceId: registryResolved.sourceId,
    endpointRole: registryResolved.endpointRole,
    credentialFingerprint256: registryResolved.credentialFingerprint256,
    registryVersion: registryResolved.registryVersion,
  },
}
```

This is the FINAL authorization. After this succeeds, the peer is fully authorized and the identity is trusted.

## Deny Precedence

**DENY / REVOCATION OVERRIDES**:
- Active registry authorization state
- Credential allowlist match
- Cutover state
- Matching network claims
- Any otherwise-successful registry resolution

**Precedence order**:
1. Installation deny (checked first)
2. Credential deny (checked second)
3. Claim matching (checked last)

**No fallback**: ANY deny hit rejects, no exceptions.

## SourceId Semantics

**SourceId semantic relationship to stationId remains OPEN.**

This slice:
- Compares `claims.source_id` against registry-derived `sourceId`
- Rejects on mismatch (`IDENTITY_MISMATCH`)
- Does NOT decide whether `sourceId` is independent or deterministically equal to `stationId`

The open requirement is preserved for future resolution.

## Certificate Authority Boundary

**Certificate-derived authority** (from B1b1b, via B1b2a):
- `installationId`
- `certificateFingerprint256`
- `serialNumber`

**Registry-derived authority** (from B1b2a):
- `endpointRole`
- `stationId`
- `centerId`
- `sourceId`
- Authorization state
- `registryVersion`

**Network claims** (from `link.hello`): NON-AUTHORITATIVE
- `station_id`, `center_id`, `source_id` are NEVER trusted as authority
- They are compared against registry-derived authoritative identity
- Mismatch → fail-closed rejection
- They NEVER select registry record, override values, or repair missing data

## Security

**Absolute deny precedence**:
- Deny is checked BEFORE claim matching
- Deny overrides valid credentials
- Deny overrides matching claims
- Deny overrides active authorization state
- No bypass, no fallback

**Fail-closed behavior**:
- Installation denied → denied
- Credential denied → denied
- Station/source mismatch → denied
- Center mismatch → denied
- Deny lookup exception → denied (caught and mapped)
- Claim comparison exception → denied (caught and mapped)
- No fallback, no repair, no claim override

**Claim validation**:
- Exact string equality (no normalization, no case-folding)
- Registry-derived values are authoritative
- Claims prove knowledge but do not grant authority
- Mismatch is treated as a security violation (fail-closed)

## Test Strategy

### Minimum test coverage

1. **Installation denied**: `installationId` in deny registry → `INSTALLATION_DENIED`
2. **Credential denied by fingerprint**: Fingerprint in deny registry → `CREDENTIAL_DENIED`
3. **Credential denied by serial**: Serial in deny registry → `CREDENTIAL_DENIED`
4. **Deny overrides valid credential**: Denied installation with valid credential → `INSTALLATION_DENIED`
5. **Deny overrides matching claims**: Denied installation with matching claims → `INSTALLATION_DENIED`
6. **Station claim mismatch**: `claims.station_id` ≠ `registryResolved.stationId` → `IDENTITY_MISMATCH`
7. **Source claim mismatch**: `claims.source_id` ≠ `registryResolved.sourceId` → `IDENTITY_MISMATCH`
8. **Center claim mismatch**: `claims.center_id` ≠ `registryResolved.centerId` → `CENTER_MISMATCH`
9. **Successful authorization**: All checks pass → `AuthorizedPeerIdentity` with all fields
10. **Final identity structure**: Verify output contains ONLY authorized fields (no extra authority)
11. **No ACTIVE transition**: This slice does not trigger ACTIVE (B1c scope)
12. **ZERO product actions**: No business handlers, no sessions, no product state

### Test doubles

Use in-memory deny registry snapshots inline with tests. No separate fixture files needed.

## Size Forecast

- **Production**: 70-90 NET
  - Deny interface/types: ~20 lines
  - Deny check logic: ~15 lines
  - Authorization composition: ~30 lines
  - Claim matching: ~15 lines
  - Error mapping: ~10 lines
- **Tests**: 90-110 NET
  - Deny scenarios: ~30 lines
  - Deny precedence: ~20 lines
  - Claim mismatch scenarios: ~25 lines
  - Successful authorization: ~15 lines
  - Boundary/ZERO product actions: ~15 lines
- **Modifications**: ~10 NET (identity.ts, contracts.ts for final type)
- **Total**: **170-210 NET**

Well within <=350 checkpoint and <=400 gate.

## Product Actions

**ZERO**

This slice performs no ACTIVE transition, no `link.accept`, no FSM composition, and no business handlers. It produces final authorization but does not activate sessions or execute product logic.
