# PR-08B1b1 Certificate Identity Extraction Design

## Decision

PR-08B1b1 adds a fail-closed, side-effect-free adapter that accepts an authenticated TLS peer, obtains its Node `crypto.X509Certificate`, parses exactly one reserved RFC 5280 URI SAN identity, and returns certificate identity evidence. It performs no registry lookup, claim comparison, link response, state transition, or product action.

The approved authority is `C:\SisAIP\openspec\changes\PR-08B1b\design.md`. This slice preserves its exact v1 URI and canonical UUIDv4 rules.

## Boundary

Included:

- Require `tlsSocket.authorized === true`.
- Obtain the peer certificate with `getPeerX509Certificate()`.
- Parse `X509Certificate.subjectAltName` using Node's documented quoted-value representation.
- Require exactly one URI in the reserved identity namespace.
- Validate exact schema `v1` and canonical lowercase UUIDv4 syntax.
- Copy `fingerprint256` and `serialNumber` from the same certificate.
- Return a typed success or typed fail-closed error without throwing.

Excluded:

- Installation registry and deny-registry access.
- Credential allowlisting.
- `stationId`, `centerId`, or `sourceId` derivation.
- Network-claim matching.
- `link.accept`, ACTIVE transition, B1c FSM, and business handlers.

## Reuse and placement

| File | Decision |
|---|---|
| `apps/desktop/electron/security/secure-link/identity.ts` | Extend the existing branded `InstallationId` identity model with `ParsedCertificateIdentity`; do not refactor existing promotion logic. |
| `apps/desktop/electron/security/secure-link/contracts.ts` | Reuse `SecureLinkResult`; no change expected. |
| `apps/desktop/electron/security/secure-link/certificate-identity.ts` | New module containing the TLS peer adapter, Node SAN tokenizer, reserved-URI classifier, exact v1 parser, and fail-closed exception boundary. |
| `apps/desktop/electron/security/transport/certificate-validator.ts` | Keep date validation unchanged; temporal validation remains an earlier transport concern. |
| `apps/desktop/electron/security/transport/tls-gateway.ts` | Keep unchanged in B1b1. Its authenticated `TLSSocket` and upgrade boundary are the B1b2 call site. |
| `apps/desktop/test/main/security/secure-link/certificate-identity.test.ts` | New focused tests for extraction, tokenization, validation, credential evidence, and boundary purity. |
| `apps/desktop/test/fixtures/tls/test-client-identity.pem` | Add one CA-signed parsing fixture with the canonical URI SAN and an unrelated SAN whose Node presentation exercises JSON quoting. No private key is required by parser tests. |

KEEP > EXTEND > REFACTOR > REPLACE is applied: existing models and result algebra are extended, while transport and prior identity promotion are not rewritten.

## Public contract

```ts
export type ParsedCertificateIdentity = Readonly<{
  schemaVersion: 1
  installationId: InstallationId
  certificateFingerprint256: string
  serialNumber: string
}>

export type CertificateIdentityError =
  | 'TLS_PEER_UNAUTHORIZED'
  | 'PEER_CERTIFICATE_MISSING'
  | 'IDENTITY_SAN_MISSING'
  | 'IDENTITY_SAN_DUPLICATE'
  | 'IDENTITY_SAN_MALFORMED'
  | 'IDENTITY_SCHEMA_UNSUPPORTED'

export type CertificateIdentityResult = SecureLinkResult<
  ParsedCertificateIdentity,
  CertificateIdentityError
>

export function extractPeerCertificateIdentity(
  peer: Pick<tls.TLSSocket, 'authorized' | 'getPeerX509Certificate'>
): CertificateIdentityResult
```

The function never throws. `TLS_PEER_UNAUTHORIZED` is returned before certificate access when `authorized` is not exactly `true`. Missing peer evidence returns `PEER_CERTIFICATE_MISSING`. Any getter, tokenizer, JSON-decoding, or parser exception returns `IDENTITY_SAN_MALFORMED`. The caller may later map all extraction failures to its local `CERT_INVALID` handling without exposing parser details to the network.

## Node parsing proof

The repository targets Node `>=20` and Electron 33 and declares `@types/node` 22. Node supplies:

- `crypto.X509Certificate`, whose `subjectAltName` is `string | undefined` and whose `fingerprint256` and `serialNumber` are read-only strings.
- `tls.TLSSocket.getPeerX509Certificate()`, returning `X509Certificate | undefined`.
- A documented SAN presentation consisting of comma-space-separated entries, each with a type prefix and colon. The value may instead be a JSON string literal when quoting is needed to avoid delimiter ambiguity introduced by values containing `, `.

No ASN.1 parser or application-level DER traversal is required. The test fixture may be PEM/DER input to Node's own `X509Certificate`; production reads only supported high-level properties.

## Tokenizer and identity parser

1. Read the SAN string once inside the fail-closed exception boundary. `undefined` or empty means `IDENTITY_SAN_MISSING`.
2. Scan left to right rather than calling `split(', ')`:
   - read a non-empty type through its first colon;
   - if the value begins with `"`, scan one complete JSON string while honoring escapes, decode it with `JSON.parse`, and require the closing quote to be followed only by end-of-input or the exact `, ` separator;
   - otherwise consume until the next `, ` separator or end-of-input;
   - reject empty entries, missing type/value, unterminated quotes, invalid JSON, trailing separators, or unexpected text after a quoted value.
3. Classify decoded entries. A value in the exact reserved namespace under a non-`URI` type is malformed. Unrelated SAN entries are ignored.
4. Count every `URI` value beginning with `urn:infoplazas:ada-nova-plus:secure-link:identity:` before validating its body. Zero is missing; two or more is duplicate, including identical, conflicting, malformed, or mixed-version duplicates.
5. For the sole reserved URI, classify a non-`v1` version component as `IDENTITY_SCHEMA_UNSUPPORTED`; then require the complete exact grammar:

```text
^urn:infoplazas:ada-nova-plus:secure-link:identity:v1:installation:([0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12})$
```

6. Return the captured UUID unchanged as `InstallationId`, schema version `1`, and the certificate's `fingerprint256` and `serialNumber` unchanged. Empty credential properties or property-access exceptions are malformed.

JSON decoding is only reversal of Node's documented presentation escaping; it is not URI normalization. The parser never trims, case-folds, Unicode-normalizes, percent-decodes, or applies generic URI equivalence.

## Data flow and B1b2 seam

```text
authorized TLSSocket
  -> getPeerX509Certificate()
  -> tokenize subjectAltName
  -> count reserved URI entries
  -> validate exact v1 URI and UUIDv4
  -> copy fingerprint256 + serialNumber
  -> ParsedCertificateIdentity
```

B1b2 can call `extractPeerCertificateIdentity(tlsSocket)` at the existing authenticated gateway boundary, then pass only a successful `ParsedCertificateIdentity` to registry/deny/credential authorization. B1b1 does not edit the gateway and therefore cannot accept WebSockets, emit protocol messages, transition state, or invoke product behavior.

## Strict TDD plan

### RED

Add failing tests first for:

- A real `X509Certificate` fixture with a valid canonical v1 URI SAN, canonical UUID, and exact fingerprint/serial result.
- Missing SAN, duplicate identical reserved URI, duplicate conflicting/mixed-version reserved URI, v2, uppercase UUID, malformed/truncated URI, leading or trailing whitespace, percent-encoded spelling, Unicode, reserved identity under a non-URI SAN type, malformed quoted JSON, and thrown peer/certificate property access.
- Unauthenticated peer and missing peer certificate.
- A quoted unrelated SAN containing `, ` before the valid identity, proving naïve comma splitting is not used.
- Exact success keys proving no station/center/source/authorization/ACTIVE/product-action result is produced.

### GREEN

Implement only the scanner, classifier, exact regex validation, typed errors, exception boundary, and credential field copy needed to satisfy those tests. Do not edit gateway or business modules.

### TRIANGULATE

Add table-driven variants for UUID version/variant bits, identical versus conflicting duplicates, unsupported versus malformed version tokens, quoted versus unquoted unrelated SANs, and URI ordering. Verify unrelated SANs are tolerated while every second reserved URI rejects.

### REFACTOR

Extract small private tokenizer/classifier helpers only after behavior is green. Keep one authoritative URI regex and namespace constant, no generic URI parser, no dependency, no normalization, no registry abstraction, and no side effects.

## Rollout and verification

The slice is additive and dormant until B1b2 invokes it. Verification is the focused Vitest file plus desktop typecheck. Rollback is deletion of the new module/test/fixture and the added output type; no persisted data or behavior migration exists.

## Size forecast

| Area | NET lines |
|---|---:|
| Production | 96 |
| Tests | 87 |
| Fixture | 21 |
| **Total** | **204** |

The 204 NET forecast passes the 400-line review gate and does not trigger the >300 stop. If implementation evidence exceeds 300 NET, stop and return to preflight before adding scope.
