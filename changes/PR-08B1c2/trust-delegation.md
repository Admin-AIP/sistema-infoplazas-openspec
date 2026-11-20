# PR-08B1c2 Trust bootstrap and delegation

Status: ACCEPTED by the maintainer on 2026-11-21; documentation only. Authorizes no runtime implementation and no
`ACTIVE` behavior. Closes the `Trust/delegation` row of `design.md` "Remaining blockers". The PKI issuance proof and the
protected Dinamizador key provisioning stay separate open blockers.

## Terms

* Trust bootstrap: how a new Usuario PC or Dinamizador knows whom to trust from its first day.
* Delegation: the institutional permission of a Dinamizador to enroll Usuario PC stations of its own center. It is
  limited to its `center_id`, to Usuario PC stations and to the automatic enrollment ceiling and approval rules of
  `production-limits.md`; it never authorizes a Dinamizador identity, another center, product actions or CA acts.
* Roles: Dinamizador is the program; Operador del Centro is the person who operates the Infoplaza; the Enlace is the human
  installer role with greater scope and privileges than the Operador.

## Decision

1. Trust root. The public root of the private CA, plus a backup root, travels inside the code-signed installer and is
   pinned locally; nobody imports it by hand. Rotating or replacing a root is done by a new installer. The code signature
   proves only the origin and integrity of the software.
2. Delegation. It is a document signed by the private CA that states the `center_id`, that it covers Usuario PC stations
   only, the delegation identifier and a version. Endpoints verify it against the pinned root. The Dinamizador receives it
   when it is enrolled and with later authorization updates when it is online.
3. No expiry. The delegation has no expiry date. It ends only when it is revoked, or when the certificate of the
   Dinamizador is revoked or replaced (a new Dinamizador identity gets its own delegation). Revocation reaches a
   Dinamizador when it is online; a center without internet keeps its delegation until then. The delegation never ends
   by itself and nothing is removed automatically.
4. First enrollment of a Dinamizador. It is automatic: no manual approval and no code to type. It needs internet once,
   because the certificate is signed by the central CA, whose signing key never sits on an endpoint, and it is done by the
   Enlace when installing. After that the Dinamizador works without internet. There is no offline provisioning package.
5. Enlace authentication. The Enlace authenticates with a cloud account and configures with internet. Actions reserved to
   the Enlace (for example changing the automatic enrollment ceiling or the technical limits) require a connection. The
   Dinamizador, the Operador and the stations keep working without internet, and the Operador may still approve extra
   stations at the center.
6. Usuario PC expectation. When a station is enrolled it records the identity and center of the Dinamizador it enrolled
   with and later accepts only that Dinamizador; trusting the CA alone is not enough.

## Accepted risk

The maintainer chose on 2026-11-21 that the first enrollment of a Dinamizador carries no proof of legitimacy (no code and no
per-installer secret). The consequence is that any equipment able to reach the issuance service could ask to be enrolled
as the Dinamizador of a center and would receive its delegation. The measures that need no extra step for the Enlace are
the audit and notice of every Dinamizador enrollment and central revocation. A proof that needs no typing, for example a
per-center installer that carries a one-time secret, was offered and declined. This risk must be reviewed again before
production deployment.

## Not decided here

The rotation cadence and the operator of the CA roots, and the custody of the backup root; the format and signing details
of the delegation document; the issuance service channel and the CSR and SPKI binding (PKI issuance blocker); the
protected certificate and key provisioning of the Dinamizador (Dinamizador keys blocker); how the cloud account of the Enlace
is bound to centers; how stations update their expectation when the Dinamizador of a center is replaced; the human-resolver authorization of
quarantine cases; runtime implementation.
