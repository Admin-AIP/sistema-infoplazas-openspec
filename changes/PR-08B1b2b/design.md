# PR-08B1b2b Deny Registry + Authorization Composition Design

## Implementation Status: SUPERSEDED

**Reason**: Mandatory size split (actual implementation: 456 NET > 450 mandatory threshold)

**Superseded by**:
- PR-08B1b2b1: Contract Bridge + Deny Evaluation (~210 NET)
- PR-08B1b2b2: Claim Binding + Final Authorization (~189 NET)

**Security architecture preserved**: The approved deny-before-claims absolute precedence, exact claim matching, and fail-closed semantics are preserved across the split.

**Original design retained below for architecture reference.**

---

## Decision

PR-08B1b2b implements deny/revocation checks and final binding authorization composition. It consumes the intermediate `RegistryResolvedInstallation` from B1b2a, applies absolute deny precedence, validates network claims against registry-authoritative identity, and produces the final `AuthorizedPeerIdentity`. It performs no registry lookup, no ACTIVE transition, and no product action.

This is the second subdivision of the original PR-08B1b2 work, mandated by the >350 NET pre-implementation checkpoint.

## Subdivision Context

**Original monolithic scope**: PR-08B1b2 (forecast: 360-450 NET, exceeded checkpoint)  
**Split reason**: Mandatory size checkpoint enforcement at >350 NET projected  
**Previous slice**: B1b2a — Registry resolution + credential matching (220-260 NET, COMPLETE + INTEGRATED)  
**This slice**: B1b2b — Deny + binding authorization (revised forecast: 187-227 NET)

**Dependency**: PR-08B1b2a MUST be integrated before B1b2b implementation begins.

## B1b2a Contract Extension

**Ownership**: This PR owns the contract bridge extension to RegistryResolvedInstallation.

**Context**: Integrated B1b2a (commit 87bfeae) propagates `credentialFingerprint256` but not `serialNumber` in its intermediate result. This PR requires certificate-derived `serialNumber` evidence to implement approved deny-by-serial functionality.

**Extended RegistryResolvedInstallation contract**:

```typescript
export type RegistryResolvedInstallation = Readonly<{
  installationId: InstallationId
  endpointRole: 'usuario-pc' | 'dinamizador'
  stationId: StationId | undefined
  centerId: CenterId
  sourceId: SourceId
  credentialFingerprint256: string
  serialNumber: string  // ← Contract bridge: certificate-derived credential evidence
  registryVersion: number
}>
```

**Authority of serialNumber**:
- **Certificate-derived** credential evidence (same authority as `credentialFingerprint256`, `installationId`)
- Propagated from `ParsedCertificateIdentity.serialNumber`
- Used by B1b2a for credential allowlist matching (OR semantics)
- Used by B1b2b for deny-by-serial checks

**Bridge scope** (modifications to B1b2a files, owned by B1b2b):
- Type definition: +1 field (`serialNumber: string`)
- Propagation: +1 line (`serialNumber: certificateIdentity.serialNumber`)
- Test updates: serialNumber expectation and propagation verification
- Size: ~17 NET bridge cost (included in revised B1b2b forecast below)

**Preservation guarantees**:
- Registry lookup remains installationId-only (unchanged)
- Endpoint role authority unchanged
- Credential allowlist semantics unchanged (OR logic preserved)
- stationId/centerId/sourceId authority unchanged
- No deny behavior introduced into B1b2a
- No final authorization in B1b2a (remains intermediate result)
- No ACTIVE transition
- No link.accept emission
- ZERO product actions in B1b2a

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
// Deny registry interface (explicit namespaces)
export interface DenyRegistry {
  checkInstallationDeny(installationId: InstallationId): Promise<boolean>
  checkFingerprintDeny(fingerprint256: string): Promise<boolean>
  checkSerialDeny(serialNumber: string): Promise<boolean>
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
const installationDenied = await denyRegistry.checkInstallationDeny(
  registryResolved.installationId
)

if (installationDenied) {
  return { ok: false, error: 'INSTALLATION_DENIED' }
}
```

**Deny precedence**: Installation deny is checked FIRST, before credential deny and before claim matching.

### Step 2: Deny by Credential

```typescript
const fingerprintDenied = await denyRegistry.checkFingerprintDeny(
  registryResolved.credentialFingerprint256
)

const serialDenied = await denyRegistry.checkSerialDeny(
  registryResolved.serialNumber
)

if (fingerprintDenied || serialDenied) {
  return { ok: false, error: 'CREDENTIAL_DENIED' }
}
```

**Deny keys** (from approved architecture):
- Installation deny: `checkInstallationDeny(installationId)`
- Fingerprint deny: `checkFingerprintDeny(fingerprint256)`
- Serial deny: `checkSerialDeny(serialNumber)`
- **ANY deny hit rejects** (no fallback)
- Explicit namespaces prevent ambiguity

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

**Revised forecast** (includes B1b2a contract bridge owned by B1b2b):

- **Production**: 87-107 NET
  - B1b2a contract bridge: ~2 insertions (serialNumber field + propagation)
  - Deny interface/types: ~20 lines
  - Deny check logic: ~15 lines (explicit checkInstallationDeny/Fingerprint/Serial)
  - Authorization composition: ~30 lines
  - Claim matching: ~15 lines
  - Error mapping: ~10 lines
- **Tests**: 115-135 NET
  - B1b2a bridge tests: ~25 lines (serialNumber propagation verification)
  - Deny scenarios: ~30 lines
  - Deny precedence: ~20 lines
  - Claim mismatch scenarios: ~25 lines
  - Successful authorization: ~15 lines
  - Boundary/ZERO product actions: ~15 lines
- **Modifications**: ~10 NET (identity.ts, contracts.ts for final type)
- **Total**: **212-252 NET**
- **Projected CHURN**: ~37 (B1b2a bridge: ~10 deletions from test expectation updates)

**Previous forecast**: 170-210 NET (B1b2b only)  
**Bridge cost**: ~17 NET (B1b2a contract extension)  
**Revised total**: 187-227 NET (combined), realistically **212-252 NET** with full test coverage

**Gate check**:  
- <=300 checkpoint: ✓ **PASS** (252 < 300)
- <=400 gate: ✓ **PASS** (252 < 400)

No subdivision required.

## Product Actions

**ZERO**

This slice performs no ACTIVE transition, no `link.accept`, no FSM composition, and no business handlers. It produces final authorization but does not activate sessions or execute product logic.
