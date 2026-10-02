# PR-08B1b1a SAN Identity Parsing Design

## Decision

PR-08B1b1a implements pure RFC 5280 subjectAltName parsing with zero TLS or certificate coupling. It tokenizes Node's documented SAN representation using quote-aware parsing, detects exactly one reserved Infoplazas identity URI, validates schema version v1, and extracts a canonical lowercase UUIDv4 installationId. It performs no TLS peer validation, no certificate metadata extraction, no registry lookup, and no product action.

This is the first subdivision of the original PR-08B1b1 work, which exceeded the mandatory 400 NET gate. The approved X.509 architecture from `changes/PR-08B1b\design.md` remains authoritative and unchanged.

## Subdivision Context

**Original monolithic scope**: PR-08B1b1 (463 NET, exceeded gate)  
**Split reason**: Mandatory size gate enforcement  
**This slice**: B1b1a — Pure SAN parsing (forecast: 328 NET)  
**Next slice**: B1b1b — Peer certificate adapter (depends on B1b1a integrated)

**Architectural decisions preserved**:
- RFC 5280 URI SAN as sole authoritative identity field
- Reserved namespace: `urn:infoplazas:ada-nova-plus:secure-link:identity:v1:installation:<UUID>`
- Canonical lowercase UUIDv4 validation (version 4, variant RFC 4122)
- No normalization (no trim, case-fold, percent-decode, Unicode-normalize, URI-equivalence)
- Fail-closed error contract
- Zero product actions

## Scope

### Included

- Accept a raw SAN string value (from any source, not limited to TLS certificates)
- Tokenize using quote-aware parsing (no naïve `split(",")`)
- Handle Node's documented JSON-quoted value escaping
- Filter entries to reserved identity namespace URIs
- Require exactly one reserved identity URI (duplicates rejected)
- Extract and validate schema version
- Enforce schema version v1 (v2+ rejected as unsupported)
- Validate canonical lowercase UUIDv4 syntax:
  - Exact format: `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`
  - Version nibble = `4`
  - Variant nibble = `8`, `9`, `a`, or `b` (RFC 4122 variant 2)
  - All hex characters lowercase
- Reject whitespace, percent encoding, Unicode, and other normalization attempts
- Return typed `ParsedSanInstallationIdentity` or typed fail-closed error

### Excluded

- `TLSSocket` access
- `X509Certificate` access
- `getPeerX509Certificate()` calls
- `peer.authorized` validation
- Certificate `fingerprint256` extraction
- Certificate `serialNumber` extraction
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
export type ParsedSanInstallationIdentity = Readonly<{
  schemaVersion: 1
  installationId: InstallationId
}>

export type SanIdentityError =
  | 'IDENTITY_SAN_MISSING'
  | 'IDENTITY_SAN_DUPLICATE'
  | 'IDENTITY_SAN_MALFORMED'
  | 'IDENTITY_SCHEMA_UNSUPPORTED'

export type SanIdentityResult = SecureLinkResult<
  ParsedSanInstallationIdentity,
  SanIdentityError
>

export function parseSanInstallationIdentity(
  subjectAltName: string | undefined
): SanIdentityResult
```

The function never throws. All tokenization, validation, or property access exceptions are caught and mapped to `IDENTITY_SAN_MALFORMED`.

## Implementation Strategy

### SAN Tokenization

Node's `X509Certificate.subjectAltName` returns a string representation:

```
TYPE:value, TYPE:value, ...
```

When a value contains `, ` (comma-space), Node JSON-encodes it:

```
TYPE:"escaped value\u002c with comma", TYPE:other
```

**Quote-aware tokenizer**:
1. Iterate character-by-character
2. Track quote state and escape sequences
3. Split on `, ` only when not inside quotes
4. After tokenization, unwrap JSON-quoted values using `JSON.parse()`
5. Return array of `TYPE:value` entries

**No naïve `split(",")` or equivalent delimiter parsing.**

### Reserved Identity Detection

Filter tokenized entries:
1. Must start with `URI:`
2. Strip `URI:` prefix
3. Check if starts with `urn:infoplazas:ada-nova-plus:secure-link:identity:`

Count filtered URIs:
- Zero → `IDENTITY_SAN_MISSING`
- Exactly one → continue
- Two or more (identical or conflicting) → `IDENTITY_SAN_DUPLICATE`

### Schema Version Validation

Check reserved identity URI prefix:
- Starts with `...identity:v1:` → valid, continue
- Starts with `...identity:v2:` or higher → `IDENTITY_SCHEMA_UNSUPPORTED`
- Starts with namespace but no recognizable version → `IDENTITY_SAN_MALFORMED`

### UUIDv4 Extraction and Validation

Expected pattern:
```
urn:infoplazas:ada-nova-plus:secure-link:identity:v1:installation:<UUID>
```

Validate with regex:
```
^urn:infoplazas:ada-nova-plus:secure-link:identity:v1:installation:([0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12})$
```

- Capture group extracts the UUID
- Version nibble must be `4`
- Variant nibble must be `8`, `9`, `a`, or `b`
- All characters must be lowercase hex

Malformed cases return `IDENTITY_SAN_MALFORMED`:
- Uppercase characters
- Wrong UUID version (1, 3, 5)
- Wrong UUID variant
- Invalid hex characters
- Missing/extra segments
- Whitespace
- Percent encoding
- Unicode characters

## TDD Requirements

Minimum test coverage:

**Happy path**:
1. Valid v1 identity URI with canonical lowercase UUIDv4 → success with `ParsedSanInstallationIdentity`

**Fail-closed**:
2. `undefined` SAN value → `IDENTITY_SAN_MISSING`
3. Empty string SAN → `IDENTITY_SAN_MISSING`
4. No reserved identity URI → `IDENTITY_SAN_MISSING`
5. Identical duplicate reserved URIs → `IDENTITY_SAN_DUPLICATE`
6. Conflicting duplicate reserved URIs → `IDENTITY_SAN_DUPLICATE`
7. Unsupported schema v2 → `IDENTITY_SCHEMA_UNSUPPORTED`
8. Uppercase UUID → `IDENTITY_SAN_MALFORMED`
9. Non-v4 UUID (version 1) → `IDENTITY_SAN_MALFORMED`
10. Malformed UUID syntax → `IDENTITY_SAN_MALFORMED`

**Tokenization**:
11. Identity URI with unrelated SAN entries → success (ignores unrelated)
12. Quoted unrelated SAN containing comma-space → correct tokenization (proves no naïve split)
13. Identity URI not first in SAN list → success (ordering independence)

## File Placement

| File | Decision |
|---|---|
| `apps/desktop/electron/security/secure-link/san-identity-parser.ts` | New module containing `parseSanInstallationIdentity()`, tokenizer, and validation logic |
| `apps/desktop/electron/security/secure-link/identity.ts` | Extend with `ParsedSanInstallationIdentity` type export; existing identity types unchanged |
| `apps/desktop/test/main/security/secure-link/san-identity-parser.test.ts` | New focused tests for SAN parsing, tokenization, validation |

## Size Forecast

| Category | Lines |
|---|---|
| Production | 133 |
| Tests | 195 |
| Fixtures | 0 |
| **Total** | **328 NET** |

**Size policy for this intentionally split slice**:
- Projected final ≤360 NET: continue
- Projected final >360 NET: STOP and report
- Actual >400 NET: mandatory STOP

No cohesion exception is pre-authorized.

## Dependencies

**Depends on**: None (pure parsing, no external integration)

**Consumed by**: PR-08B1b1b (peer certificate adapter imports this parser)

## Product Actions

**ZERO**

No registry lookup, no authorization, no state transition, no business logic.

## Next Slice

**PR-08B1b1b** consumes `parseSanInstallationIdentity()` and adds:
- TLS peer validation (`peer.authorized`)
- Certificate retrieval (`getPeerX509Certificate()`)
- Certificate metadata extraction (`fingerprint256`, `serialNumber`)
- Final `ParsedCertificateIdentity` output

B1b1b depends on B1b1a integrated first.
