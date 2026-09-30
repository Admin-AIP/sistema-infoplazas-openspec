# PR-08B1a2a2 — WSS Upgrade / Gateway Lifecycle
## Production Architecture Design

**Change ID**: PR-08B1a2a2  
**Scope**: WebSocket Secure (WSS) upgrade infrastructure + certificate temporal validation  
**Target**: ADA NOVA PLUS / Dinamizador Desktop  
**Baselines**:
- Dinamizador master: 4cc81696b0b8f9de4868029f54b7abbb9197b9ae
- Policy A (PR-08B1a2a1r): CLOSED (r1+r2 complete)

---

## EXECUTIVE SUMMARY

This change extends the TLS Gateway with WebSocket Secure (WSS) upgrade capability and integrates certificate temporal validation. It maintains strict transport security ordering: TLS 1.3 handshake → mTLS → Policy A (resumed session rejection) → **certificate temporal validation** → upgrade validation → WebSocket connection establishment.

**Key Architectural Decisions**:
1. **WSS Server Pattern**: `WebSocketServer({ noServer: true })` with `httpsServer.on('upgrade', ...)` hook
2. **Temporal Validation Placement**: During upgrade event, BEFORE `handleUpgrade`, using `TLSSocket.getPeerCertificate()`
3. **Rejection Semantics**: Failed validations destroy socket immediately without upgrading
4. **Lifecycle Contract**: WebSocketServer created/destroyed with gateway, upgrade listener prevents new connections during shutdown
5. **Dependency**: `ws@^8.18.0` runtime, `@types/ws@^8.5.13` dev

**What B1a2a2 Does NOT Include**:
- X.509 business identity validation (installationId/stationId/centerId) → B1b
- Connection/payload limits, timeouts, rate limiting → B1a2b
- Message handling, FSM composition, product actions → B1c
- Server certificate provisioning → separate infrastructure
- Clock plausibility validation → separate concern (temporal validation uses system clock as-is)

**Size Forecast**: ~320 lines changed (within 400-line budget)
- Production: ~110 lines (tls-gateway.ts + 2 new functions)
- Tests: ~200 lines (upgrade scenarios, temporal validation, lifecycle)
- Dependencies: +2 entries (package.json)

---

## PHASE 1 — TRANSPORT SECURITY ORDERING

### 1.1 Complete Transport Stack

```
┌─────────────────────────────────────────────────────────────┐
│ Application Layer (NOT B1a2a2)                              │
│ - Message validation, FSM, product actions                  │
│ - Connection limits, timeouts, rate limiting                │
│ - Business identity (installationId/stationId/centerId)     │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ WebSocket frames
                          │
┌─────────────────────────────────────────────────────────────┐
│ B1a2a2 SCOPE: WSS Upgrade Infrastructure                    │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Step 6: handleUpgrade                                   │ │
│ │   - WebSocket handshake (Sec-WebSocket-Accept)          │ │
│ │   - Connection event with ws.WebSocket instance         │ │
│ └─────────────────────────────────────────────────────────┘ │
│                          ▲                                   │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Step 5: Upgrade Validation (B1a2a2)                     │ │
│ │   - HTTP method === 'GET'                               │ │
│ │   - Header: Upgrade: websocket                          │ │
│ │   - Header: Connection: Upgrade                         │ │
│ │   - Header: Sec-WebSocket-Version: 13                   │ │
│ │   → Reject if malformed (400 + destroy)                 │ │
│ └─────────────────────────────────────────────────────────┘ │
│                          ▲                                   │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Step 4: Certificate Temporal Validation (B1a2a2 + B1a1) │ │
│ │   - Extract peer cert: socket.getPeerCertificate()      │ │
│ │   - Parse notBefore/notAfter from valid_from/valid_to   │ │
│ │   - Call validateCertificateDates(dates, now)           │ │
│ │   → Reject CERT_NOT_YET_VALID or CERT_EXPIRED           │ │
│ │   → Destroy socket BEFORE handleUpgrade                 │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ HTTP Upgrade request
                          │
┌─────────────────────────────────────────────────────────────┐
│ Existing Infrastructure (Pre-B1a2a2)                        │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Step 3: Policy A — Resumed Session Rejection            │ │
│ │   - Executes in secureConnection listener               │ │
│ │   - enforceFreshTlsConnection(tlsSocket)                │ │
│ │   → Destroy if isSessionReused() === true               │ │
│ └─────────────────────────────────────────────────────────┘ │
│                          ▲                                   │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Step 2: Mutual TLS (mTLS) Validation                    │ │
│ │   - requestCert: true, rejectUnauthorized: true         │ │
│ │   - ca: trustedCA (private CA only)                     │ │
│ │   → Reject if !tlsSocket.authorized                     │ │
│ └─────────────────────────────────────────────────────────┘ │
│                          ▲                                   │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Step 1: TLS 1.3 Handshake                               │ │
│ │   - minVersion/maxVersion: TLSv1.3                      │ │
│ │   - secureOptions: SSL_OP_NO_TICKET                     │ │
│ │   → Reject TLS < 1.3                                    │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ TCP/TLS
                          │
                    [Client Socket]
```

### 1.2 Critical Ordering Invariants

1. **TLS Handshake Precedes All Application Logic**: No HTTP data parsed until TLS 1.3 establishes
2. **Policy A Enforced Before HTTP Parsing**: `secureConnection` listener fires before HTTP `request`/`upgrade`
3. **Temporal Validation During Upgrade**: Certificate dates validated in `upgrade` event, using peer certificate extracted from TLSSocket
4. **Upgrade Validation Before handleUpgrade**: Malformed upgrade requests rejected without WebSocket handshake
5. **Shutdown Prevents New Upgrades**: `upgrade` listener checks shutdown state early

### 1.3 Execution Flow for Valid WSS Connection

```
Client initiates TLS 1.3 handshake with client certificate
    ↓
TLS 1.3 negotiation completes (fresh session only, no resumption)
    ↓
secureConnection event fires
    ↓
enforceFreshTlsConnection() checks isSessionReused()
    ↓ (false = fresh)
tlsSocket.authorized check passes (mTLS valid)
    ↓
Client sends HTTP Upgrade request
    ↓
upgrade event fires
    ↓
Check gateway not shutting down
    ↓
Extract peer certificate: socket.getPeerCertificate()
    ↓
Parse valid_from → notBefore, valid_to → notAfter
    ↓
validateCertificateDates({ notBefore, notAfter, now: new Date() })
    ↓ (ok: true)
Validate upgrade headers (GET, Upgrade, Connection, Sec-WebSocket-Version)
    ↓
wss.handleUpgrade(req, socket, head, (ws) => { wss.emit('connection', ws, req) })
    ↓
connection event fires → WebSocket ready for messages
```

---

## PHASE 2 — WSS SERVER ARCHITECTURE

### 2.1 WebSocketServer Pattern

**Pattern**: `{ noServer: true }` with manual upgrade handling

**Rationale**:
- HTTPS server already bound to port/host for TLS
- `noServer: true` allows WebSocketServer without independent HTTP server
- Manual `handleUpgrade` enables transport security checks BEFORE WebSocket handshake
- Existing `secureConnection` listener remains authoritative for TLS/mTLS enforcement

**Constructor**:
```typescript
import { WebSocketServer } from 'ws'

const wss = new WebSocketServer({ noServer: true })
```

**Lifecycle Binding**:
- Created during `start()` after HTTPS server starts
- Destroyed during `stop()` before HTTPS server closes
- No independent port binding

### 2.2 Upgrade Event Hook

**Placement**: After HTTPS server starts, before gateway becomes addressable

```typescript
server.on('upgrade', (req, socket, head) => {
  // Step 1: Check shutdown state
  if (!wss) {
    socket.destroy()
    return
  }

  // Step 2: Extract TLS socket
  const tlsSocket = socket as tls.TLSSocket

  // Step 3: Certificate temporal validation (B1a2a2 integration point)
  const peerCert = tlsSocket.getPeerCertificate()
  if (!peerCert || Object.keys(peerCert).length === 0) {
    // Missing peer certificate (should not happen after mTLS, but defensive)
    socket.destroy()
    return
  }

  const certValidation = validatePeerCertificateDates(peerCert)
  if (!certValidation.ok) {
    // Reject CERT_NOT_YET_VALID or CERT_EXPIRED
    socket.destroy()
    return
  }

  // Step 4: Upgrade request validation
  const upgradeValidation = validateUpgradeRequest(req)
  if (!upgradeValidation.ok) {
    socket.write(
      'HTTP/1.1 400 Bad Request\r\n' +
      'Content-Type: text/plain\r\n' +
      '\r\n' +
      upgradeValidation.error
    )
    socket.destroy()
    return
  }

  // Step 5: Perform WebSocket handshake
  wss.handleUpgrade(req, socket, head, (ws) => {
    wss.emit('connection', ws, req)
  })
})
```

**Ordering Guarantees**:
- `upgrade` event fires AFTER `secureConnection` (Policy A already enforced)
- Certificate temporal validation runs BEFORE `handleUpgrade`
- Upgrade validation runs BEFORE `handleUpgrade`
- Socket destroyed immediately on any validation failure

### 2.3 Certificate Temporal Validation Integration

**New Function**: `validatePeerCertificateDates(peerCert: tls.PeerCertificate)`

**Purpose**: Extract temporal fields from TLSSocket peer certificate and delegate to existing `validateCertificateDates`

**Implementation Contract**:
```typescript
import type * as tls from 'node:tls'
import { validateCertificateDates } from './certificate-validator'

type PeerCertDateValidationResult =
  | Readonly<{ ok: true }>
  | Readonly<{ ok: false; error: 'CERT_NOT_YET_VALID' | 'CERT_EXPIRED' | 'CERT_DATES_UNPARSEABLE' }>

function validatePeerCertificateDates(
  peerCert: tls.PeerCertificate
): PeerCertDateValidationResult {
  // Extract temporal fields from peer certificate
  const { valid_from, valid_to } = peerCert

  if (!valid_from || !valid_to) {
    return { ok: false, error: 'CERT_DATES_UNPARSEABLE' }
  }

  // Parse ISO 8601 strings to Date objects
  const notBefore = new Date(valid_from)
  const notAfter = new Date(valid_to)

  // Validate parsed dates
  if (isNaN(notBefore.getTime()) || isNaN(notAfter.getTime())) {
    return { ok: false, error: 'CERT_DATES_UNPARSEABLE' }
  }

  // Delegate to existing validator
  return validateCertificateDates({
    notBefore,
    notAfter,
    now: new Date()
  })
}
```

**TLSSocket API Contract** (Node.js documented behavior):
- `tlsSocket.getPeerCertificate()` returns `PeerCertificate` object
- `valid_from`: ISO 8601 string (e.g., "Jan 1 00:00:00 2024 GMT")
- `valid_to`: ISO 8601 string (e.g., "Jan 1 00:00:00 2025 GMT")
- Empty object `{}` if no peer certificate (should not occur after mTLS, but defensive)

**Error Semantics**:
- `CERT_DATES_UNPARSEABLE`: Missing or malformed temporal fields → destroy socket
- `CERT_NOT_YET_VALID`: Certificate not yet valid → destroy socket
- `CERT_EXPIRED`: Certificate expired → destroy socket

**Clock Plausibility**: NOT validated in B1a2a2. System clock used as-is. Separate concern for future.

### 2.4 Upgrade Request Validation

**New Function**: `validateUpgradeRequest(req: http.IncomingMessage)`

**Purpose**: Validate HTTP Upgrade request compliance with WebSocket protocol (RFC 6455)

**Implementation Contract**:
```typescript
import type * as http from 'node:http'

type UpgradeValidationResult =
  | Readonly<{ ok: true }>
  | Readonly<{ ok: false; error: string }>

function validateUpgradeRequest(req: http.IncomingMessage): UpgradeValidationResult {
  // Method MUST be GET
  if (req.method !== 'GET') {
    return { ok: false, error: 'Method must be GET' }
  }

  // Upgrade header MUST be present and contain "websocket"
  const upgradeHeader = req.headers['upgrade']
  if (!upgradeHeader || !upgradeHeader.toLowerCase().includes('websocket')) {
    return { ok: false, error: 'Missing or invalid Upgrade header' }
  }

  // Connection header MUST be present and contain "Upgrade"
  const connectionHeader = req.headers['connection']
  if (!connectionHeader || !connectionHeader.toLowerCase().includes('upgrade')) {
    return { ok: false, error: 'Missing or invalid Connection header' }
  }

  // Sec-WebSocket-Version MUST be 13 (RFC 6455)
  const versionHeader = req.headers['sec-websocket-version']
  if (versionHeader !== '13') {
    return { ok: false, error: 'Unsupported WebSocket version' }
  }

  return { ok: true }
}
```

**Validation Rules**:
1. HTTP method: `GET` (required by RFC 6455)
2. `Upgrade` header: must contain `"websocket"` (case-insensitive)
3. `Connection` header: must contain `"Upgrade"` (case-insensitive)
4. `Sec-WebSocket-Version` header: must be `"13"` (current WebSocket protocol version)

**Rejection Behavior**:
- Send `HTTP/1.1 400 Bad Request` with plain-text error
- Destroy socket immediately
- No WebSocket handshake performed

**Path Routing**: NOT in B1a2a2 scope. All valid upgrade requests accepted. Future slices may add path-based routing.

---

## PHASE 3 — LIFECYCLE SEMANTICS

### 3.1 Start Sequence

```typescript
async start(): Promise<void> {
  // 1. Guard: prevent double start
  if (server) {
    throw new Error('Gateway already started')
  }

  // 2. Create HTTPS server (existing)
  server = https.createServer({ /* TLS config */ }, (req, res) => {
    res.writeHead(200)
    res.end()
  })

  // 3. Bind existing listeners (secureConnection, error)
  server.on('secureConnection', (tlsSocket) => {
    if (!enforceFreshTlsConnection(tlsSocket)) {
      return
    }
    if (!tlsSocket.authorized) {
      tlsSocket.destroy()
    }
  })

  // 4. **NEW**: Create WebSocketServer
  wss = new WebSocketServer({ noServer: true })

  // 5. **NEW**: Bind upgrade listener
  server.on('upgrade', (req, socket, head) => {
    // Implementation from Phase 2
  })

  // 6. Start HTTPS server (existing)
  await new Promise<void>((resolve, reject) => {
    server!.listen(config.port, config.host, () => {
      const addr = server!.address()
      if (addr && typeof addr === 'object') {
        boundAddress = { host: addr.address, port: addr.port }
        resolve()
      } else {
        reject(new Error('Failed to get server address'))
      }
    })
    server!.once('error', (err) => reject(err))
  })

  // 7. Bind runtime error handler (existing)
  server.on('error', (err) => {
    console.error('[TLS Gateway] Runtime error:', err)
  })
}
```

**Order Invariants**:
1. HTTPS server created first (owns TLS/mTLS)
2. WebSocketServer created BEFORE server starts listening (prevent race)
3. `upgrade` listener bound BEFORE server starts listening (prevent lost upgrades)
4. Server listens last (gateway becomes addressable only when fully configured)

### 3.2 Stop Sequence

```typescript
async stop(): Promise<void> {
  // 1. Guard: allow idempotent stop
  if (!server) {
    return
  }

  // 2. **NEW**: Close WebSocketServer
  //    Prevents new 'connection' events, but does NOT close active WebSockets
  //    (ws library behavior: close() stops accepting new connections)
  if (wss) {
    wss.close()
    wss = null
  }

  // 3. Close HTTPS server
  //    Stops accepting new TCP connections
  //    Waits for existing connections to complete or timeout
  await new Promise<void>((resolve, reject) => {
    server!.close((err) => {
      if (err) reject(err)
      else resolve()
    })
  })

  // 4. Nullify state
  server = null
  boundAddress = null
}
```

**Shutdown Guarantees**:
1. WebSocketServer closed BEFORE HTTPS server (prevents new WebSocket connections)
2. Active WebSocket connections NOT forcibly terminated (graceful drain)
3. HTTPS `server.close()` waits for all connections to complete (Node.js behavior)
4. Upgrade listener implicitly disabled via `wss` nullification (guard in upgrade handler)

**Double Stop Safety**: Idempotent (no-op if already stopped)

**Active Socket Handling**:
- B1a2a2 does NOT forcibly close active WebSocket connections during shutdown
- Connections drain naturally (client closes or connection timeout)
- Future slices may add timeout-based forced closure

### 3.3 Restart Behavior

**Scenario**: `stop()` followed by `start()`

**Guarantees**:
1. New `server` instance created (old instance fully released)
2. New `wss` instance created (old instance released)
3. New `upgrade` listener bound (no duplication)
4. No listener leaks (Node.js event emitter adds listeners to new server instance)

**Test Coverage Required**:
- Restart sequence (start → stop → start)
- Verify no duplicate upgrade events
- Verify new connections work after restart

### 3.4 Upgrade During Shutdown

**Scenario**: Client sends upgrade request between `wss.close()` and `server.close()` completion

**Behavior**:
```typescript
server.on('upgrade', (req, socket, head) => {
  if (!wss) {  // wss set to null during stop()
    socket.destroy()
    return
  }
  // ... rest of upgrade logic
})
```

**Guarantee**: Socket destroyed immediately, no WebSocket handshake performed

---

## PHASE 4 — DEPENDENCY JUSTIFICATION

### 4.1 ws Library

**Package**: `ws@^8.18.0` (runtime), `@types/ws@^8.5.13` (dev)

**Justification**:
- De facto standard WebSocket library for Node.js
- Mature, actively maintained (17k+ stars, 1.8B+ downloads/month)
- Supports `noServer: true` pattern for manual upgrade handling
- Zero native dependencies (pure JavaScript with optional C++ acceleration)
- Compatible with Electron 33 (Node 20 embedded) and Node 24 (CLI tests)

**Alternatives Rejected**:
- Native Node.js WebSocket API: Not available in Node 20/24 (experimental in Node 21+)
- `websocket` package: Less maintained, heavier dependency tree
- `ws-server`: Abandoned, unmaintained

**Security Posture**:
- Well-audited (used in production by major projects: Socket.io, Next.js, etc.)
- No CVEs in last 2 years (as of latest npm audit)
- Actively patched when vulnerabilities discovered

### 4.2 Compatibility Matrix

| Environment | Node Version | ws Support | Status |
|-------------|--------------|------------|--------|
| Electron 33 (embedded) | 20.x | ✅ Full | Primary runtime |
| CLI tests (Vitest) | 24.x | ✅ Full | Test environment |
| Future Electron upgrades | 22.x+ | ✅ Full | Forward compatible |

**TypeScript Types**:
- `@types/ws@^8.5.13` provides complete type definitions
- Matches runtime version `ws@^8.x`
- No type/runtime version mismatch

---

## PHASE 5 — TDD PROOF TABLE

### 5.1 Transport Security Proof (Existing + New)

| Test Case | Policy A | mTLS | TLS 1.3 | Temporal | Upgrade | Expected Outcome |
|-----------|----------|------|---------|----------|---------|------------------|
| Valid fresh TLS 1.3 + mTLS + valid dates + valid upgrade | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | **WSS connection established** |
| Resumed TLS session | ❌ Fail | N/A | N/A | N/A | N/A | Socket destroyed (Policy A) |
| TLS 1.2 connection | N/A | N/A | ❌ Fail | N/A | N/A | Handshake rejected |
| Missing client certificate | N/A | ❌ Fail | N/A | N/A | N/A | Socket destroyed (mTLS) |
| Untrusted client certificate | N/A | ❌ Fail | N/A | N/A | N/A | Socket destroyed (mTLS) |
| Certificate not yet valid | ✅ Pass | ✅ Pass | ✅ Pass | ❌ CERT_NOT_YET_VALID | N/A | **Socket destroyed BEFORE upgrade** |
| Certificate expired | ✅ Pass | ✅ Pass | ✅ Pass | ❌ CERT_EXPIRED | N/A | **Socket destroyed BEFORE upgrade** |
| Certificate dates unparseable | ✅ Pass | ✅ Pass | ✅ Pass | ❌ CERT_DATES_UNPARSEABLE | N/A | **Socket destroyed BEFORE upgrade** |
| Certificate at exact notBefore | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | WSS connection established |
| Certificate at exact notAfter | ✅ Pass | ✅ Pass | ✅ Pass | ❌ CERT_EXPIRED | N/A | Socket destroyed BEFORE upgrade |
| Valid transport, invalid HTTP method (POST) | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Method | HTTP 400 + socket destroyed |
| Valid transport, missing Upgrade header | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Header | HTTP 400 + socket destroyed |
| Valid transport, missing Connection header | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Header | HTTP 400 + socket destroyed |
| Valid transport, invalid WebSocket version | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Version | HTTP 400 + socket destroyed |

**Proof Metrics**:
- Upgrade count MUST be zero for all rejected connections
- Upgrade count MUST be exactly 1 for accepted connections
- Connection count MUST be exactly 1 for successful upgrades

### 5.2 Lifecycle Proof

| Test Case | Expected Behavior | Proof Metric |
|-----------|-------------------|--------------|
| Start gateway | Server listens, WSS ready | No errors, address() returns host/port |
| Double start | Throws error | Error message: "Gateway already started" |
| Stop gateway | Server closes, WSS closed | No errors, address() returns null |
| Double stop | No-op, no error | Idempotent behavior |
| Restart (start → stop → start) | New server/WSS instances | New connections accepted after restart |
| Upgrade during shutdown | Socket destroyed | No WebSocket connection event |
| Listener leak check (10 restarts) | No duplicate listeners | EventEmitter listener count stable |

### 5.3 Certificate Temporal Validation Proof

| Test Case | notBefore | notAfter | now | Expected Result |
|-----------|-----------|----------|-----|-----------------|
| Valid range (mid-period) | 2024-01-01 | 2025-01-01 | 2024-06-15 | ✅ ok: true |
| Valid range (at notBefore) | 2024-06-15 12:00:00 | 2025-01-01 | 2024-06-15 12:00:00 | ✅ ok: true |
| Not yet valid (before notBefore) | 2025-01-01 | 2026-01-01 | 2024-06-15 | ❌ CERT_NOT_YET_VALID |
| Expired (after notAfter) | 2023-01-01 | 2024-01-01 | 2024-06-15 | ❌ CERT_EXPIRED |
| Expired (at notAfter) | 2023-01-01 | 2024-06-15 12:00:00 | 2024-06-15 12:00:00 | ❌ CERT_EXPIRED |
| Missing valid_from | undefined | 2025-01-01 | 2024-06-15 | ❌ CERT_DATES_UNPARSEABLE |
| Missing valid_to | 2024-01-01 | undefined | 2024-06-15 | ❌ CERT_DATES_UNPARSEABLE |
| Malformed date string | "invalid" | 2025-01-01 | 2024-06-15 | ❌ CERT_DATES_UNPARSEABLE |

**Integration Proof**: All temporal validation test cases MUST execute BEFORE `handleUpgrade` (verified by counting upgrade events).

### 5.4 Scope Boundary Proof (What Doesn't Happen)

| Test Case | Expected Behavior | Proof Metric |
|-----------|-------------------|--------------|
| Valid WSS connection established | No product actions triggered | Product action count = 0 |
| Valid WSS connection established | No business identity validation | No installationId/stationId checks |
| Valid WSS connection established | No message payload validation | No message parsing/validation |
| Valid WSS connection established | No connection limit enforcement | No max connections check |
| Valid WSS connection established | No rate limiting | No request throttling |

**Proof**: B1a2a2 ONLY establishes WSS transport. No application-layer logic.

---

## PHASE 6 — API SURFACE CHANGES

### 6.1 TlsGateway Interface (Extended)

**No Breaking Changes**: Existing interface preserved, internal implementation extended

```typescript
// UNCHANGED PUBLIC INTERFACE
export interface TlsGatewayConfig {
  readonly serverCert: Buffer
  readonly serverKey: Buffer
  readonly trustedCA: Buffer
  readonly host: string
  readonly port: number
}

export interface TlsGateway {
  start(): Promise<void>
  stop(): Promise<void>
  address(): { host: string; port: number } | null
}
```

**Internal Changes**:
- `start()`: Creates WebSocketServer, binds upgrade listener (INTERNAL)
- `stop()`: Closes WebSocketServer before server (INTERNAL)

**External Consumers**: No changes required (backward compatible)

### 6.2 New Exported Functions

```typescript
/**
 * Validate peer certificate temporal validity during upgrade
 * @param peerCert - TLS peer certificate from socket.getPeerCertificate()
 * @returns Validation result with ok/error
 * @internal Exported for testing only, not public API
 */
export function validatePeerCertificateDates(
  peerCert: tls.PeerCertificate
): PeerCertDateValidationResult

/**
 * Validate HTTP Upgrade request compliance with WebSocket protocol
 * @param req - HTTP IncomingMessage from upgrade event
 * @returns Validation result with ok/error
 * @internal Exported for testing only, not public API
 */
export function validateUpgradeRequest(
  req: http.IncomingMessage
): UpgradeValidationResult
```

**Export Rationale**: Enables isolated unit testing of validation logic without full integration test overhead. Marked `@internal` to signal non-public API.

### 6.3 No Connection Event Exposure (Yet)

**B1a2a2 Scope**: Gateway accepts WSS connections but does NOT expose them to application layer

**Future Extension** (B1c or later):
```typescript
// NOT IN B1a2a2
export interface TlsGateway {
  // ... existing methods
  onConnection(handler: (ws: WebSocket) => void): void  // Future: B1c
}
```

**Proof**: Tests verify upgrade succeeds, but do NOT consume WebSocket connection events

---

## PHASE 7 — FILE CHANGE MANIFEST

### 7.1 Production Files

| File | Change Type | Lines Changed | Description |
|------|-------------|---------------|-------------|
| `apps/desktop/electron/security/transport/tls-gateway.ts` | Modified | ~110 | Add WSS server, upgrade listener, validation functions |
| `apps/desktop/package.json` | Modified | +2 | Add `ws` runtime + `@types/ws` dev dependencies |

**Total Production**: ~112 lines changed

### 7.2 Test Files

| File | Change Type | Lines Changed | Description |
|------|-------------|---------------|-------------|
| `apps/desktop/test/main/security/transport/tls-gateway.test.ts` | Modified | ~200 | Add WSS upgrade tests, temporal validation tests, lifecycle tests |

**Total Tests**: ~200 lines changed

### 7.3 No New Files

All changes are extensions to existing files. No new modules introduced.

---

## PHASE 8 — SIZE ESTIMATE & DELIVERY GATE

### 8.1 Size Breakdown

| Category | Lines Changed |
|----------|---------------|
| Production code | 112 |
| Test code | 200 |
| **Total** | **312** |

**Gate Status**: ✅ **PASS** (≤400 lines)

### 8.2 Single PR Viability

**Recommendation**: Single PR delivery

**Rationale**:
- Size well within 400-line budget
- Semantic coherence: WSS upgrade + temporal validation are tightly coupled
- No natural split point (both validation functions serve upgrade handler)
- Test coverage integrated (upgrade tests validate temporal validation)

**Risk Assessment**: Low
- No breaking changes (backward compatible)
- Isolated to transport layer (no business logic)
- Comprehensive test coverage prevents regressions

---

## PHASE 9 — RISK ANALYSIS & MITIGATION

### 9.1 Certificate Date Parsing Risk

**Risk**: `valid_from`/`valid_to` string format varies across OpenSSL versions or certificate types

**Mitigation**:
1. Defensive parsing with `isNaN()` checks
2. Return `CERT_DATES_UNPARSEABLE` error (explicit failure mode)
3. Test coverage includes malformed date scenarios
4. Production logging (future) for unparseable dates to detect edge cases

**Impact**: Low (Node.js TLSSocket API is stable, consistent format)

### 9.2 WebSocketServer Shutdown Race

**Risk**: Active WebSocket connection receives data between `wss.close()` and full shutdown

**Mitigation**:
1. B1a2a2 does NOT handle messages (race cannot affect unimplemented logic)
2. Future slices (B1c) must implement message handlers defensively (check connection state)
3. Test coverage verifies upgrade prevented during shutdown

**Impact**: None in B1a2a2 (deferred to B1c)

### 9.3 Clock Skew / Temporal Validation Bypass

**Risk**: System clock manipulation allows expired certificates to validate

**Mitigation**:
1. NOT in B1a2a2 scope (clock plausibility is separate concern)
2. Future: Clock plausibility service (NTP validation, monotonic clock checks)
3. Certificate temporal validation still provides defense-in-depth (prevents accidental expiry)

**Impact**: Accepted risk (documented as out-of-scope)

### 9.4 Dependency Supply Chain

**Risk**: `ws` package compromised or abandoned

**Mitigation**:
1. Pin to specific version (`ws@^8.18.0`, not `*` or `latest`)
2. Regular `npm audit` in CI pipeline
3. Dependency update policy (security patches only, major versions reviewed)
4. Fallback: `ws` is mature, stable, widely used (low abandonment risk)

**Impact**: Low (mitigated by version pinning + audit process)

---

## PHASE 10 — IMPLEMENTATION GUIDANCE

### 10.1 TDD Workflow

1. **Test First**: Write temporal validation tests (unit + integration)
2. **Red**: Tests fail (validation functions not implemented)
3. **Green**: Implement `validatePeerCertificateDates` and `validateUpgradeRequest`
4. **Refactor**: Extract common error handling patterns
5. **Integration**: Add upgrade listener to gateway `start()`
6. **Lifecycle Tests**: Verify start/stop/restart semantics
7. **Scope Proof**: Verify product action count = 0

### 10.2 Certificate Test Data Generation

**Approach**: Use existing test certificate infrastructure (from Policy A tests)

**Test Certificates Required**:
- Valid certificate (notBefore < now < notAfter)
- Not-yet-valid certificate (notBefore > now)
- Expired certificate (notAfter < now)
- Boundary certificates (now === notBefore, now === notAfter)

**Generation**: Extend existing certificate generation helpers (if needed)

### 10.3 Integration Test Pattern

**Scenario**: Real TLS 1.3 mTLS client performs WSS upgrade

```typescript
it('accepts valid WSS upgrade with fresh TLS and valid certificate dates', async () => {
  // Arrange
  const gateway = createTlsGateway(config)
  await gateway.start()

  const clientCert = generateValidCertificate() // notBefore < now < notAfter

  // Act
  const client = await createTlsClient({
    cert: clientCert.cert,
    key: clientCert.key,
    ca: trustedCA,
    rejectUnauthorized: true
  })

  await client.upgradeToWebSocket('/') // Send GET with Upgrade headers

  // Assert
  expect(client.isWebSocketConnected()).toBe(true)
  expect(gateway.upgradeCount).toBe(1) // Proof: upgrade succeeded
  expect(gateway.productActionCount).toBe(0) // Proof: no business logic
})
```

### 10.4 Shutdown Test Pattern

```typescript
it('rejects upgrade during shutdown', async () => {
  // Arrange
  const gateway = createTlsGateway(config)
  await gateway.start()

  const client = await createTlsClient(validConfig)

  // Act
  const stopPromise = gateway.stop() // Begin shutdown
  const upgradePromise = client.upgradeToWebSocket('/') // Race upgrade

  // Assert
  await expect(upgradePromise).rejects.toThrow() // Socket destroyed
  await stopPromise // Clean shutdown completes
  expect(gateway.upgradeCount).toBe(0) // No upgrade succeeded
})
```

---

## PHASE 11 — EXPLICIT SCOPE BOUNDARIES (RECAP)

### 11.1 What B1a2a2 DOES Include

✅ WebSocketServer creation with `noServer: true` pattern  
✅ `httpsServer.on('upgrade', ...)` listener binding  
✅ Certificate temporal validation during upgrade (using `validateCertificateDates`)  
✅ Peer certificate extraction from TLSSocket (`getPeerCertificate()`)  
✅ Upgrade request validation (method, headers, version)  
✅ Socket destruction on validation failures (BEFORE `handleUpgrade`)  
✅ Gateway lifecycle (start/stop) with WebSocketServer integration  
✅ Shutdown prevents new upgrades  
✅ Restart safety (no listener leaks)  
✅ `ws` + `@types/ws` dependency addition  

### 11.2 What B1a2a2 DOES NOT Include

❌ X.509 business identity validation (installationId/stationId/centerId) → B1b  
❌ WebSocket message handling, parsing, or validation → B1c  
❌ Connection limits, max connections, concurrent connection enforcement → B1a2b  
❌ Payload size limits, message size validation → B1a2b  
❌ Connection timeouts, idle timeouts, ping/pong keepalive → B1a2b  
❌ Rate limiting, request throttling → B1a2b  
❌ FSM composition, state machine integration → B1c  
❌ Product action dispatch, business logic → B1c  
❌ Session management, session store → B1c  
❌ Server certificate provisioning, certificate rotation → separate infrastructure  
❌ Clock plausibility validation, NTP synchronization → separate concern  
❌ Path-based routing, multi-endpoint support → future enhancement  
❌ Subprotocol negotiation (Sec-WebSocket-Protocol) → future enhancement  

---

## PHASE 12 — DEPENDENCY PACKAGE.JSON CHANGES

### 12.1 Runtime Dependency

```json
{
  "dependencies": {
    "ws": "^8.18.0"
  }
}
```

**Justification**: WebSocket protocol implementation (see Phase 4)

### 12.2 Development Dependency

```json
{
  "devDependencies": {
    "@types/ws": "^8.5.13"
  }
}
```

**Justification**: TypeScript type definitions for `ws` package

### 12.3 Version Constraints

- `^8.18.0`: Caret range allows patch + minor updates (8.18.x, 8.19.x, etc.)
- Excludes breaking major version 9.x (when released)
- Security patches automatically included via `npm update`

---

## PHASE 13 — VERIFICATION CHECKLIST

### 13.1 Pre-Implementation Verification

- [ ] Design reviewed and approved
- [ ] Scope boundaries confirmed (no B1a2b/B1c absorption)
- [ ] Size estimate confirms ≤400 lines
- [ ] TDD proof table covers all security invariants
- [ ] Dependency justification accepted

### 13.2 Implementation Verification

- [ ] All TDD proof table tests implemented and passing
- [ ] Certificate temporal validation integrated BEFORE `handleUpgrade`
- [ ] Policy A rejection prevents upgrade (existing test still passes)
- [ ] TLS 1.2 / missing cert / untrusted cert rejected (existing tests still pass)
- [ ] Upgrade validation rejects malformed requests
- [ ] Lifecycle tests (start/stop/restart) passing
- [ ] Listener leak test passing (10 restarts)
- [ ] Product action count = 0 for all tests
- [ ] TypeScript compilation clean (no type errors)

### 13.3 Pre-Merge Verification

- [ ] All tests passing (new + existing)
- [ ] No regressions in Policy A behavior
- [ ] Code review completed
- [ ] Size within budget (≤400 lines actual)
- [ ] Documentation updated (if public API changed)

---

## DESIGN AUTHORITY

This design is authoritative for PR-08B1a2a2 implementation. Deviations require explicit design amendment with rationale.

**Sign-off Required**: Architecture design approved before implementation phase

**Next Phase**: Implementation (execute TDD workflow, deliver single PR)

---

## APPENDIX A — WEBSOCKET PROTOCOL REFERENCE

### RFC 6455 Upgrade Requirements

**Client Request**:
```
GET / HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**Server Response** (from `handleUpgrade`):
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**B1a2a2 Responsibility**: Validate client request, delegate handshake response to `ws` library

### ws Library API Contract

```typescript
import { WebSocketServer } from 'ws'

const wss = new WebSocketServer({ noServer: true })

server.on('upgrade', (req, socket, head) => {
  wss.handleUpgrade(req, socket, head, (ws) => {
    wss.emit('connection', ws, req)
  })
})

wss.on('connection', (ws, req) => {
  // WebSocket ready for messages (NOT in B1a2a2)
})
```

---

## APPENDIX B — POLICY A INTEGRATION PROOF

### Existing Policy A Behavior (Preserved)

**Test**: Resumed TLS session rejected BEFORE upgrade

```typescript
it('rejects resumed TLS session (Policy A) before upgrade attempt', async () => {
  // Arrange
  const gateway = createTlsGateway(config)
  await gateway.start()

  const client = await createTlsClient(validConfig)
  await client.connect() // First connection (fresh)
  await client.disconnect()

  // Act: Attempt resumed session
  await client.connect({ resumeSession: true })

  // Assert
  expect(client.isConnected()).toBe(false) // Socket destroyed by Policy A
  expect(gateway.upgradeCount).toBe(0) // Upgrade never reached
})
```

**Guarantee**: Policy A enforcement (`enforceFreshTlsConnection`) remains authoritative. Upgrade listener only executes for fresh TLS sessions.

---

**END OF ARCHITECTURE DESIGN**
