# PR-08B1b1b Peer Certificate Identity Extraction Design

## Decision

PR-08B1b1b implements a fail-closed TLS peer adapter that validates `peer.authorized`, obtains the peer's `X509Certificate`, consumes the pure SAN parser from PR-08B1b1a, and extracts certificate credential metadata (`fingerprint256`, `serialNumber`). It produces the final `ParsedCertificateIdentity` type required for secure-link identity flow. It performs no registry lookup, no authorization decision, and no product action.

This is the second subdivision of the original PR-08B1b1 work. The approved X.509 architecture from `changes/PR-08B1b\design.md` remains authoritative and unchanged.

## Subdivision Context

**Original monolithic scope**: PR-08B1b1 (463 NET, exceeded gate)  
**Split reason**: Mandatory size gate enforcement  
**Previous slice**: B1b1a — Pure SAN parsing (328 NET, must be integrated first)  
**This slice**: B1b1b — Peer certificate adapter (forecast: 152 NET)

**Dependency**: PR-08B1b1a **MUST** be integrated before B1b1b implementation begins.

## Scope

### Included

- Accept authenticated `TLSSocket` seam
- Validate `peer.authorized === true`
- Call `peer.getPeerX509Certificate()` with exception handling
- Reject missing peer certificate
- Extract `cert.subjectAltName` with exception handling
- **Consume** `parseSanInstallationIdentity()` from B1b1a
- Propagate `installationId` from B1b1a result
- Extract `cert.fingerprint256` with exception handling
- Extract `cert.serialNumber` with exception handling
- Produce `ParsedCertificateIdentity` with all fields
- Map adapter/property exceptions fail-closed
- Extend (not replace) B1b1a error codes with TLS-specific errors

### Excluded

- SAN tokenization (delegated to B1b1a)
- UUID validation (delegated to B1b1a)
- Duplicate detection (delegated to B1b1a)
- Schema version validation (delegated to B1b1a)
- Installation registry lookup
- Deny registry
- Credential allowlisting
- `stationId`, `centerId`, or `sourceId` derivation or authorization
- Network claim matching
- `link.accept`, ACTIVE transition, B1c FSM
- Business handlers
- Product actions: **ZERO**

## Public Contract

```typescript
export type ParsedCertificateIdentity = Readonly<{
  schemaVersion: 1
  installationId: InstallationId
  certificateFingerprint256: string
  serialNumber: string
}>

export type CertificateIdentityError =
  | 'TLS_PEER_UNAUTHORIZED'
  | 'PEER_CERTIFICATE_MISSING'
  | SanIdentityError  // Reuses B1b1a errors: IDENTITY_SAN_MISSING, IDENTITY_SAN_DUPLICATE, IDENTITY_SAN_MALFORMED, IDENTITY_SCHEMA_UNSUPPORTED

export type CertificateIdentityResult = SecureLinkResult<
  ParsedCertificateIdentity,
  CertificateIdentityError
>

export function extractPeerCertificateIdentity(
  peer: Pick<TLSSocket, 'authorized' | 'getPeerX509Certificate'>
): CertificateIdentityResult
```

The function never throws. All adapter exceptions are caught and mapped fail-closed.

## Implementation Strategy

### Step 1: TLS Peer Authorization Check

```typescript
if (peer.authorized !== true) {
  return { ok: false, error: 'TLS_PEER_UNAUTHORIZED' }
}
```

This check happens **before** certificate access. It prevents processing of untrusted peers.

### Step 2: Certificate Retrieval

```typescript
let cert: X509Certificate | undefined
try {
  cert = peer.getPeerX509Certificate()
} catch {
  return { ok: false, error: 'PEER_CERTIFICATE_MISSING' }
}

if (!cert) {
  return { ok: false, error: 'PEER_CERTIFICATE_MISSING' }
}
```

### Step 3: SAN Extraction

```typescript
let subjectAltName: string | undefined
try {
  subjectAltName = cert.subjectAltName
} catch {
  return { ok: false, error: 'IDENTITY_SAN_MALFORMED' }
}
```

### Step 4: Delegate to B1b1a Parser

```typescript
const sanResult = parseSanInstallationIdentity(subjectAltName)

if (!sanResult.ok) {
  return sanResult  // Propagate B1b1a error unchanged
}

const { installationId } = sanResult.value
```

**No re-parsing, no re-validation**. B1b1a owns all SAN logic.

### Step 5: Certificate Metadata Extraction

```typescript
let fingerprint256: string
let serialNumber: string

try {
  fingerprint256 = cert.fingerprint256
  serialNumber = cert.serialNumber

  if (!fingerprint256 || !serialNumber) {
    return { ok: false, error: 'PEER_CERTIFICATE_MISSING' }
  }
} catch {
  return { ok: false, error: 'PEER_CERTIFICATE_MISSING' }
}
```

### Step 6: Return Final Identity

```typescript
return {
  ok: true,
  value: {
    schemaVersion: 1,
    installationId,
    certificateFingerprint256: fingerprint256,
    serialNumber,
  },
}
```

## Error Propagation

B1b1a errors propagate **unchanged**:
- `IDENTITY_SAN_MISSING`
- `IDENTITY_SAN_DUPLICATE`
- `IDENTITY_SAN_MALFORMED`
- `IDENTITY_SCHEMA_UNSUPPORTED`

B1b1b adds adapter-layer errors:
- `TLS_PEER_UNAUTHORIZED` (before certificate access)
- `PEER_CERTIFICATE_MISSING` (missing cert or missing metadata)

The caller may later map all extraction failures to local handling (e.g., `CERT_INVALID`) without exposing parser details to the network.

## TDD Requirements

Minimum test coverage:

**Adapter layer**:
1. Unauthorized TLS peer → `TLS_PEER_UNAUTHORIZED`
2. Missing peer certificate → `PEER_CERTIFICATE_MISSING`
3. `getPeerX509Certificate()` throws → `PEER_CERTIFICATE_MISSING`
4. Certificate missing `fingerprint256` → `PEER_CERTIFICATE_MISSING`
5. Certificate missing `serialNumber` → `PEER_CERTIFICATE_MISSING`

**Happy path**:
6. Valid authenticated peer with valid B1b1a SAN → `ParsedCertificateIdentity` with all fields

**B1b1a error propagation**:
7. B1b1a returns `IDENTITY_SAN_MISSING` → propagated unchanged
8. B1b1a returns `IDENTITY_SAN_MALFORMED` → propagated unchanged

**Boundary verification**:
9. Output contains `schemaVersion: 1`
10. Output contains `installationId` from B1b1a
11. Output contains `certificateFingerprint256` from certificate
12. Output contains `serialNumber` from certificate
13. Output does **NOT** contain `stationId`, `centerId`, or `sourceId`

## File Placement

| File | Decision |
|---|---|
| `apps/desktop/electron/security/secure-link/certificate-identity.ts` | New module containing `extractPeerCertificateIdentity()`, imports B1b1a parser |
| `apps/desktop/electron/security/secure-link/identity.ts` | Extend with `ParsedCertificateIdentity` type export; existing types unchanged |
| `apps/desktop/test/main/security/secure-link/certificate-identity.test.ts` | New focused tests for peer adapter, certificate extraction, boundary verification |

## Size Forecast

| Category | Lines |
|---|---|
| Production | 68 |
| Tests | 84 |
| Fixtures | 0 |
| **Total** | **152 NET** |

**Size policy**:
- Target: ≤400 NET (248-line margin)
- Checkpoint: if projected >300 NET, verify before continuing
- Absolute gate: ≤400 NET

## Dependencies

**Depends on**: 
- **PR-08B1b1a** (must be integrated first)
  - Imports `parseSanInstallationIdentity()`
  - Reuses `SanIdentityError` codes
  - Extends `ParsedSanInstallationIdentity` to `ParsedCertificateIdentity`

**Consumed by**: 
- **PR-08B1b2** (authorization logic, future work)

## Relationship to B1b2

B1b1b **stops** at identity extraction. It does **NOT** implement:
- Installation registry lookup
- `stationId` authorization
- `centerId` authorization  
- `sourceId` authorization
- Deny registry check
- Credential allowlist check
- Network claim matching
- `link.accept` response
- ACTIVE transition

Those responsibilities belong to **PR-08B1b2**, which depends on B1b1b integrated.

## Product Actions

**ZERO**

No registry, no authorization, no state transition, no business logic.

## Implementation Readiness

After B1b1a is integrated to Dinamizador `origin/master`, B1b1b implementation may begin.

**Authoritative base for B1b1b**: Dinamizador `origin/master` with B1b1a integrated (future commit, TBD after B1b1a review and integration).
