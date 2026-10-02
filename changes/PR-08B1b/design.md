PR-08B1b X.509 IDENTITY DESIGN

IDENTITY MODEL

installationId:
- meaning: Globally stable identifier of one enrolled software/machine installation and its authorization lineage; it is not a hardware serial, hostname, MAC address, logical station, center, person, certificate serial, or public-key fingerprint. Ordinary certificate renewal/reissuance and an explicitly authorized key cutover retain it; reinstall, replacement, cloning recovery, or creation of another installation receives a new value.
- owner: Infoplazas installation-enrollment authority and authoritative installation registry; the private CA only binds the authority-approved value from the authorized CSR/provisioning transaction.
- uniqueness: Global across all Usuario PC and Dinamizador installations. Version 1 values are cryptographically random RFC 4122/RFC 9562 UUIDv4 values, canonical lowercase, with collision rejection by a unique registry constraint.
- mutability: Immutable for the lifetime of the installation identity and unchanged across certificate renewal/reissuance; a lifecycle event that creates a new installation creates a new installationId.
- certificate-bound: YES
- reason: It is the minimum immutable business principal that mTLS must bind cryptographically. The registry can then authorize that principal and derive mutable assignments without trusting network claims.

stationId:
- meaning: Stable logical Usuario PC/workstation identity used by inventory, session, and single-active-link rules; it survives certificate renewal and may survive reinstall or hardware replacement.
- owner: Authoritative station inventory/registration service, projected into the local authorized-installation registry.
- uniqueness: Global across logical Usuario PC stations. Registry ingestion accepts only canonical 1–64 character lowercase ASCII identifiers matching `^[a-z0-9](?:[a-z0-9._-]{0,62}[a-z0-9])?$` and enforces uniqueness.
- mutability: The identifier itself is immutable; the installation-to-station assignment is mutable only through an explicit replacement/cutover transaction. It MUST NOT change silently during a connection.
- certificate-bound: NO
- reason: It identifies a logical station rather than the credential-bearing installation. Binding it in every certificate would duplicate registry state and force certificate reissuance for legitimate replacement/cutover operations. It is trusted only after lookup from the certificate-bound installationId.

centerId:
- meaning: Stable identifier of one Infoplaza/center, not a display name, address, hostname, or network location.
- owner: Authoritative Infoplazas center registry, projected into each endpoint's local authorization registry.
- uniqueness: Global across centers. Registry ingestion uses the same canonical 1–64 character lowercase ASCII identifier grammar as stationId and enforces uniqueness.
- mutability: The center identifier is immutable, but an installation/station center assignment is mutable by an explicit, monotonic registry update. It MUST remain fixed for one authorization decision/connection.
- certificate-bound: NO
- reason: Center assignment is operational state. Deriving it from the authorized local installation record avoids stale center data in long-lived certificates and permits reassignment without certificate reissuance. Reassignment still requires disabling the old binding and distributing the updated local registry/deny state; fully offline nodes can enforce only their latest locally known state.

SELECTED X.509 ENCODING

- selected mechanism: One authoritative URI GeneralName in Subject Alternative Name (SAN); Subject CN/O/OU, DNS SANs, custom extensions, and all duplicate/hybrid identity representations are non-authoritative and ignored for business identity.
- exact field/extension: RFC 5280 `subjectAltName` (`2.5.29.17`), exactly one `GeneralName.uniformResourceIdentifier` in the reserved Infoplazas secure-link namespace. Criticality follows the RFC 5280 CA profile (including critical SAN when the Subject is empty) and does not change application enforcement.
- exact syntax: SAN URI value `urn:infoplazas:ada-nova-plus:secure-link:identity:v1:installation:<installationId>`, where `<installationId>` matches lowercase UUIDv4 `^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`. It is ASCII-only IA5String. No query, fragment, percent escape, whitespace, Unicode, alternate slash form, trailing delimiter, or URI-equivalent spelling is accepted.
- schema version: Explicit `v1` path component inside the reserved URI namespace. A certificate contains exactly one reserved-namespace identity URI and therefore exactly one schema version. Future versions use `...:identity:vN:...`; migration is by reader support plus certificate reissuance, never by placing v1 and v2 identities together. installationId remains stable on ordinary reissuance.
- canonicalization: None at runtime. Issuance emits the one canonical lowercase representation; extraction performs exact case-sensitive ASCII grammar validation and returns the captured UUID unchanged. It MUST NOT trim, case-fold, Unicode-normalize, percent-decode, apply generic URI equivalence, or infer identity from CN/O/OU/DNS. Registry-derived stationId, centerId, and sourceId are likewise validated at ingestion and claims are compared by exact string equality without normalization.
- duplicate handling: Any second SAN URI in the reserved `urn:infoplazas:ada-nova-plus:secure-link:identity:` namespace—identical, conflicting, supported, or unsupported—makes the certificate identity ambiguous and is rejected. Unrelated SAN entries MAY exist but confer no authority. Conflicting values in CN/O/OU, otherName, or custom extensions are ignored rather than merged.
- malformed handling: Fail closed on malformed ASN.1/SAN presentation, invalid Node SAN quoting/tokenization, wrong GeneralName type, wrong prefix/order, non-ASCII data, bad UUID, forbidden encoding, or parser exception. Use Node's `tls.TLSSocket.getPeerX509Certificate()`/raw DER and `crypto.X509Certificate.subjectAltName`; a dedicated tokenizer MUST implement Node's documented JSON-quoted SAN presentation rules before classifying entries, never naïvely split or regex-search inside quoted values. It then applies the exact grammar and does no URI decoding.
- missing handling: A path-valid certificate with no SAN, no reserved identity URI, or no usable raw/X509 certificate is unauthorized and the socket is closed before identity-dependent negotiation or privilege.
- unsupported version handling: Any reserved-namespace identity whose sole version is not exactly `v1` is rejected as an unsupported certificate-identity schema. There is no fallback to CN/O/OU, another URI, a registry guess, or a lower version.
- reason selected: URI SAN is a standard RFC 5280 GeneralName, ASCII and namespaceable, supported by common CA/CSR tooling, visible through supported Node X.509 APIs, deterministic under an exact grammar, offline-capable, and simpler than adding an ASN.1 dependency. CN is legacy, semantically overloaded, escaped, and repeatable; O/OU are mutable organizational DNs and repeatable; DNS SAN introduces DNS case/IDNA/trailing-dot semantics and misstates a non-host identity; otherName/custom OID would be strongly typed but require an allocated OID plus structured ASN.1 tooling not exposed by Node's high-level API; a hybrid duplicates authority and creates downgrade/conflict rules. URI SAN therefore gives the best security/operational balance, provided the Node-formatted SAN tokenizer is strict and DER fixtures prove duplicate, quoted, malformed, and unsupported-version behavior.

CERTIFICATE VS REGISTRY

- certificate carries: The immutable installationId in the single v1 SAN URI, plus ordinary X.509 issuer, serial, public key, validity, EKU, and fingerprints used as credential evidence. It does not carry stationId, centerId, sourceId, authorization state, deny state, display metadata, or current assignment.
- registry provides: A locally available, durable, internally consistent record keyed uniquely by installationId: endpoint role (`usuario-pc` or `dinamizador`), stationId when applicable, centerId, canonical sourceId, authorization status, record/version lineage, and the active certificate/SPKI fingerprint or serial allowlist needed to distinguish renewal/cutover credentials. Exact overlap is allowed only for an explicit cutover and deny wins over allow.
- center binding source: The latest locally trusted authorized-installation registry record. A reassignment is a monotonic transaction that disables the old binding and publishes the new binding; it does not reinterpret an in-flight connection. Certificate reissuance is unnecessary solely for a center change, although credential rotation MAY be operationally required for a physical transfer. An isolated old center can know only its last synchronized state, the already-approved offline revocation limitation.
- deny/revocation source: Local durable deny/revocation registry synchronized from the authority when possible, indexed by installationId and exact credential fingerprint/serial. It is checked before allow/overlap status and remains effective offline. TLS chain revocation and business deny are separate controls; either one denies.

B1b EXTRACTION CONTRACT

- input: A freshly authenticated, non-resumed TLS connection with `tlsSocket.authorized === true`; its peer `crypto.X509Certificate`/raw DER; explicit current time/temporal result from B1a1; expected endpoint role; a consistent local authorized-installation registry snapshot; a local deny/revocation snapshot; and, at the later claim-binding call boundary, untrusted `source_id`, `station_id`, and `center_id` from `link.hello`.
- output: A discriminated result, never nullable authority. Success returns `ParsedCertificateIdentity { schemaVersion: 1, installationId, certificateFingerprint256, serialNumber }`, followed by one immutable `AuthorizedPeerIdentity { installationId, stationId?, centerId, sourceId, endpointRole, credentialFingerprint256, registryVersion }`. Denial returns a typed internal reason and no trusted identity. B1b does not transition to ACTIVE and performs zero product action.
- failure modes: Missing peer certificate; TLS/path/temporal failure; missing, malformed, ambiguous, duplicated, or unsupported certificate identity; unknown installation; duplicate/corrupt/inconsistent registry records; wrong endpoint role; unauthorized/disabled installation; credential not in the active/explicit-overlap set; installation or credential in deny/revocation; missing station/center/source binding; and exact hello claim mismatch. `station_id`/`source_id` mismatch is `IDENTITY_MISMATCH`; `center_id` mismatch is `CENTER_MISMATCH`. Certificate/registry failures before a safely correlatable hello are locally audited and closed without inventing `link.reject`; denied/revoked credentials use local `CERT_REVOKED`, unknown/unauthorized installation uses local `AUTH_REQUIRED`, and malformed certificate identity uses local `CERT_INVALID` detail.
- fail-closed behavior: Every parser exception, unknown state, missing dependency, unsupported version, duplicate, registry inconsistency, deny hit, or mismatch returns denied, grants no trusted identity, sends no `link.accept`, performs no ACTIVE transition, and executes zero product action. Only certificate-derived installationId may select the registry record; network claims, CN/O/OU, IP, MAC, hostname, mDNS, and display names never select or repair identity. Audit records only trusted identifiers established before the failure.

SECURITY

- ambiguity defenses: Exactly one reserved SAN URI; one explicit version; one strict grammar; duplicate reserved entries reject even if equal; no CN/O/OU/DNS/custom-extension fallback; no hybrid precedence; registry uniqueness and role checks; exact credential fingerprint/serial allowlisting prevents an otherwise CA-signed replacement certificate from inheriting authorization merely by copying installationId.
- normalization defenses: ASCII-only canonical identifiers, lowercase UUIDv4, exact case-sensitive comparisons, no trim/case-fold/Unicode normalization/IDNA/percent-decoding/URI equivalence, and a Node-aware quoted-SAN tokenizer prevent Unicode, delimiter, quoted-value, and alternate-spelling attacks. Malformed ASN.1 or tokenizer uncertainty fails closed.
- reassignment handling: centerId and station assignment live only in the registry. A monotonic reassignment disables the old binding, updates the new binding, invalidates existing authorization state, and requires reconnect/re-evaluation. No stale center remains embedded in the certificate. Offline sites necessarily enforce their last known local state; high-risk physical transfer should rotate the credential and deny the old credential, but immediate global revocation cannot be promised while isolated.
- unauthorized CA-signed certificate handling: Valid CA signature, dates, and SAN syntax are necessary but insufficient. The installation must exist, have the expected role/status/bindings, and present an explicitly active or cutover credential fingerprint/serial. Unknown, mis-issued, or same-installationId/different-key certificates are denied.
- revocation/deny handling: TLS/path revocation when locally available and business deny are independent fail-closed gates. Deny is evaluated by installation and exact credential before allow; it overrides authorization and cutover overlap. A byte-for-byte cloned certificate plus copied private key is cryptographically indistinguishable offline, so prevention depends on non-exportable protected keys and response depends on locally synchronized deny/revocation.

PR-08B1b IMPACT

- architecture ready after approval: YES — approval of the listed maintainer decisions closes the X.509 encoding blocker; implementation still requires a fresh apply preflight and strict TDD.
- estimated production NET: 190–250 across a new certificate-identity parser, local installation/deny authorization module, focused `identity.ts` model/promotion changes, and narrow TLS-gateway/context integration.
- estimated test NET: 260–360 for DER certificate fixtures or fixture generation, strict SAN parser cases, registry/deny/role/credential authorization, claim mismatches, and real-TLS fail-closed integration.
- estimated total NET: 450–610, excluding OpenSpec documentation and any production provisioning UI/storage adapter.
- subdivision required: YES under the 400-line review budget. `ask-on-risk` requires a maintainer delivery choice before apply; the natural seam is certificate identity extraction/fixtures first, then registry/deny/center/claim authorization and gateway integration, but no chain strategy or exception is selected by this design.
- product actions: ZERO

OPEN QUESTIONS REMAINING:
- No X.509 field, syntax, canonicalization, center-placement, duplicate, malformed, missing, or versioning ambiguity remains if this design is approved.
- Confirm the existing `sourceId` business meaning and whether it is an independent registry field or deterministically equal to stationId; in either case it is registry-derived, never certificate-derived or claim-authoritative.
- Production PKI tooling must demonstrate issuance of the exact URI SAN from an authorized CSR/provisioning transaction and rejection of requester-supplied unauthorized identity values; physical protected-key provisioning remains the separate recorded open requirement.
- The registry synchronization/high-water mechanism must preserve monotonic center reassignment and deny updates; its transport/storage implementation is outside this encoding decision.
- Skill resolution was `fallback-path`: no parent-injected phase-skill path was supplied, so the explicit Gentle AI and cognitive-document design skill paths were used without registry discovery.

MAINTAINER DECISIONS REQUIRED:
- Approve the authoritative SAN URI namespace and canonical UUIDv4 installationId contract, including exact rejection rather than normalization.
- Approve registry-derived stationId, centerId, and sourceId, plus credential fingerprint/serial allowlisting and deny precedence.
- Approve the documented offline reassignment/revocation limitation and the recommendation to rotate credentials for high-risk physical transfers.
- Under `ask-on-risk`, choose the delivery strategy for the forecast 450–610 NET change: approve a bounded subdivision and chain strategy, or explicitly accept a separately authorized size exception; no exception is currently approved.
