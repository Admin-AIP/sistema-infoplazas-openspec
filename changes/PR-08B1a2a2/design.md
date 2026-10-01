# PR-08B1a2a2 — WSS Upgrade / Gateway Lifecycle (CORRECTED)
## Production Architecture Design

**Change ID**: PR-08B1a2a2  
**Scope**: WebSocket Secure (WSS) upgrade infrastructure + certificate temporal validation  
**Target**: ADA NOVA PLUS / Dinamizador Desktop  
**Baselines**:
- Dinamizador master: 4cc81696b0b8f9de4868029f54b7abbb9197b9ae
- Policy A (PR-08B1a2a1r): CLOSED (r1+r2 complete)
- Original design: commit 4af6d660

**Correction Pass**: Architecture corrections for lifecycle, testability, and validation semantics

---

## EXECUTIVE SUMMARY

This change extends the TLS Gateway with WebSocket Secure (WSS) upgrade capability and integrates certificate temporal validation. It maintains strict transport security ordering: TLS 1.3 handshake → mTLS → Policy A (resumed session rejection) → **certificate temporal validation** → upgrade validation → WebSocket connection establishment.

**Key Architectural Decisions**:
1. **WSS Server Pattern**: `WebSocketServer({ noServer: true })` with `httpsServer.on('upgrade', ...)` hook
2. **Temporal Validation Placement**: During upgrade event, BEFORE `handleUpgrade`, using `TLSSocket.getPeerCertificate()`
3. **Rejection Semantics**: Failed validations destroy socket immediately without upgrading
4. **Lifecycle Contract**: WebSocketServer created/destroyed with gateway, **explicit client tracking + termination during shutdown**
5. **Dependency**: `ws@^8.18.0` runtime, `@types/ws@^8.5.13` dev

**What B1a2a2 Does NOT Include**:
- X.509 business identity validation (installationId/stationId/centerId) → B1b
- Connection/payload limits, timeouts, rate limiting → B1a2b
- Message handling, FSM composition, product actions → B1c
- Server certificate provisioning → separate infrastructure
- Clock plausibility validation → separate concern (temporal validation uses system clock as-is)

**Size Forecast**: ~372 lines changed (within 400-line budget)
- Production: ~140 lines (tls-gateway.ts + lifecycle state + client tracking)
- Tests: ~220 lines (upgrade scenarios, temporal validation, lifecycle, synthetic adapter tests)
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
│ │   - TRACK CLIENT in active connections Set              │ │
│ └─────────────────────────────────────────────────────────┘ │
│                          ▲                                   │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Step 5: Upgrade Validation (B1a2a2)                     │ │
│ │   - HTTP method === 'GET'                               │ │
│ │   - Header: Upgrade contains TOKEN "websocket"          │ │
│ │   - Header: Connection contains TOKEN "upgrade"         │ │
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
5. **Shutdown Prevents New Upgrades**: `upgrade` listener checks shutdown state early (lifecycle state = SHUTTING_DOWN)
6. **Client Tracking**: Every successful WebSocket connection tracked in Set, explicitly terminated during stop()

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
Check gateway lifecycle state !== SHUTTING_DOWN
    ↓
Extract peer certificate: socket.getPeerCertificate()
    ↓
Parse valid_from → notBefore, valid_to → notAfter
    ↓
validateCertificateDates({ notBefore, notAfter, now: new Date() })
    ↓ (ok: true)
Validate upgrade headers (GET, token-based Upgrade/Connection, Sec-WebSocket-Version)
    ↓
wss.handleUpgrade(req, socket, head, (ws) => { wss.emit('connection', ws, req) })
    ↓
connection event fires → TRACK CLIENT in activeConnections Set
    ↓
WebSocket ready for messages
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
- Destroyed during `stop()` AFTER explicitly terminating all tracked clients
- No independent port binding

### 2.2 Upgrade Event Hook

**Placement**: After HTTPS server starts, before gateway becomes addressable

```typescript
server.on('upgrade', (req, socket, head) => {
  // Step 1: Check shutdown state (CORRECTION 1: lifecycle state check)
  if (lifecycleState !== 'RUNNING' || !wss) {
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

  // Step 4: Upgrade request validation (CORRECTION 4: token-based matching)
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
    // CORRECTION 1: Track active client
    activeConnections.add(ws)
    
    // Remove from tracking when closed
    ws.once('close', () => {
      activeConnections.delete(ws)
    })
    
    wss.emit('connection', ws, req)
  })
})
```

**Ordering Guarantees**:
- `upgrade` event fires AFTER `secureConnection` (Policy A already enforced)
- Certificate temporal validation runs BEFORE `handleUpgrade`
- Upgrade validation runs BEFORE `handleUpgrade`
- Socket destroyed immediately on any validation failure
- **NEW**: Active clients tracked in Set for deterministic shutdown

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

  // CORRECTION 3: Parse Node date format (NOT ISO 8601)
  // Format: "Aug 14 00:00:00 2017 GMT"
  // Defensive parsing handles variations across Node versions
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
- `valid_from`: **Date-time string** (e.g., "Aug 14 00:00:00 2017 GMT") — **NOT ISO 8601**
- `valid_to`: **Date-time string** (e.g., "Aug 14 00:00:00 2025 GMT") — **NOT ISO 8601**
- Empty object `{}` if no peer certificate (should not occur after mTLS, but defensive)

**Error Semantics**:
- `CERT_DATES_UNPARSEABLE`: Missing or malformed temporal fields → destroy socket
- `CERT_NOT_YET_VALID`: Certificate not yet valid → destroy socket
- `CERT_EXPIRED`: Certificate expired → destroy socket

**Clock Plausibility**: NOT validated in B1a2a2. System clock used as-is. Separate concern for future.

**Node Version Compatibility**: Defensive parsing verified in Node 20 (Electron 33 embedded) and Node 24 (CLI tests).

### 2.4 Upgrade Request Validation

**New Function**: `validateUpgradeRequest(req: http.IncomingMessage)`

**Purpose**: Validate HTTP Upgrade request compliance with WebSocket protocol (RFC 6455)

**Implementation Contract** (CORRECTION 4: Token-based matching):
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

  // Upgrade header MUST contain TOKEN "websocket" (case-insensitive)
  // CORRECTION 4: Proper token matching, NOT substring includes()
  const upgradeHeader = req.headers['upgrade']
  if (!upgradeHeader || !containsToken(upgradeHeader, 'websocket')) {
    return { ok: false, error: 'Missing or invalid Upgrade header' }
  }

  // Connection header MUST contain TOKEN "upgrade" (case-insensitive)
  // CORRECTION 4: Proper token matching, NOT substring includes()
  const connectionHeader = req.headers['connection']
  if (!connectionHeader || !containsToken(connectionHeader, 'upgrade')) {
    return { ok: false, error: 'Missing or invalid Connection header' }
  }

  // Sec-WebSocket-Version MUST be 13 (RFC 6455)
  const versionHeader = req.headers['sec-websocket-version']
  if (versionHeader !== '13') {
    return { ok: false, error: 'Unsupported WebSocket version' }
  }

  // Sec-WebSocket-Key: Delegated to ws.handleUpgrade (RFC 6455 base64 validation)
  // ws library validates key presence and format during handleUpgrade
  
  return { ok: true }
}

/**
 * Check if a comma-separated header contains a specific token
 * Token matching per RFC 2616 (case-insensitive, trimmed)
 */
function containsToken(headerValue: string, token: string): boolean {
  const tokens = headerValue.split(',').map(t => t.trim().toLowerCase())
  return tokens.includes(token.toLowerCase())
}
```

**Validation Rules**:
1. HTTP method: `GET` (required by RFC 6455)
2. `Upgrade` header: must contain TOKEN `"websocket"` (case-insensitive, proper comma/token semantics)
3. `Connection` header: must contain TOKEN `"upgrade"` (case-insensitive, proper comma/token semantics)
4. `Sec-WebSocket-Version` header: must be `"13"` (current WebSocket protocol version)
5. `Sec-WebSocket-Key` header: **Delegated to `ws.handleUpgrade`** (RFC 6455 requires base64-encoded 16-byte value; ws library validates during handshake)

**Token Matching Rationale** (CORRECTION 4):
- `includes('websocket')` incorrectly matches "notwebsocket"
- `containsToken()` splits on comma, trims whitespace, performs case-insensitive exact match
- Prevents false positives while handling multi-value headers correctly

**Rejection Behavior**:
- Send `HTTP/1.1 400 Bad Request` with plain-text error
- Destroy socket immediately
- No WebSocket handshake performed

**Path Routing**: NOT in B1a2a2 scope. All valid upgrade requests accepted. Future slices may add path-based routing.

---

## PHASE 3 — LIFECYCLE SEMANTICS (CORRECTION 1: EXPLICIT CLIENT TRACKING)

### 3.1 Lifecycle States

**State Machine**:
```
STOPPED ──start()──> RUNNING ──stop()──> SHUTTING_DOWN ──(clients terminated)──> STOPPED
   ▲                                                                                │
   └────────────────────────────────────────────────────────────────────────────────┘
```

**State Definitions**:
- `STOPPED`: Gateway not running, no server/WSS instances
- `RUNNING`: Accepting new TCP connections, TLS handshakes, and WebSocket upgrades
- `SHUTTING_DOWN`: Rejecting new upgrades, actively terminating tracked clients, closing servers

**State Enforcement**:
```typescript
type LifecycleState = 'STOPPED' | 'RUNNING' | 'SHUTTING_DOWN'

let lifecycleState: LifecycleState = 'STOPPED'
let server: https.Server | null = null
let wss: WebSocketServer | null = null
let activeConnections: Set<WebSocket> = new Set()
```

### 3.2 Start Sequence

```typescript
async start(): Promise<void> {
  // 1. Guard: prevent double start
  if (lifecycleState !== 'STOPPED') {
    throw new Error('Gateway already started or shutting down')
  }

  // 2. Transition to RUNNING (early, before listeners)
  lifecycleState = 'RUNNING'
  activeConnections = new Set()

  // 3. Create HTTPS server (existing)
  server = https.createServer({ /* TLS config */ }, (req, res) => {
    res.writeHead(200)
    res.end()
  })

  // 4. Bind existing listeners (secureConnection, error)
  server.on('secureConnection', (tlsSocket) => {
    if (!enforceFreshTlsConnection(tlsSocket)) {
      return
    }
    if (!tlsSocket.authorized) {
      tlsSocket.destroy()
    }
  })

  // 5. **NEW**: Create WebSocketServer
  wss = new WebSocketServer({ noServer: true })

  // 6. **NEW**: Bind upgrade listener (with lifecycle state check)
  server.on('upgrade', (req, socket, head) => {
    // Implementation from Phase 2 (checks lifecycleState === 'RUNNING')
  })

  // 7. Start HTTPS server (existing)
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

  // 8. Bind runtime error handler (existing)
  server.on('error', (err) => {
    console.error('[TLS Gateway] Runtime error:', err)
  })
}
```

**Order Invariants**:
1. Lifecycle state set to RUNNING FIRST (before any listeners bind)
2. HTTPS server created (owns TLS/mTLS)
3. WebSocketServer created BEFORE server starts listening (prevent race)
4. `upgrade` listener bound BEFORE server starts listening (prevent lost upgrades)
5. Server listens last (gateway becomes addressable only when fully configured)

### 3.3 Stop Sequence (CORRECTION 1: DETERMINISTIC CLIENT TERMINATION)

**Policy Decision**: **Option A (PREFERRED)** — Immediate termination without graceful close timeout

**Rationale**:
- Deterministic behavior (no timeout machinery)
- Simplest correct implementation
- No scope leakage into B1a2b (timeouts/limits domain)
- Restart guarantee: ZERO old sockets survive

```typescript
async stop(): Promise<void> {
  // 1. Guard: allow idempotent stop, prevent double stop during shutdown
  if (lifecycleState === 'STOPPED') {
    return // Idempotent: already stopped
  }
  
  if (lifecycleState === 'SHUTTING_DOWN') {
    throw new Error('Gateway already shutting down')
  }

  // 2. **CORRECTION 1**: Transition to SHUTTING_DOWN FIRST
  //    This prevents new upgrade attempts from succeeding
  lifecycleState = 'SHUTTING_DOWN'

  // 3. **CORRECTION 1**: Stop listening for new TCP connections
  //    BEFORE terminating clients (no new connections during shutdown)
  if (server) {
    server.removeAllListeners('upgrade')
    server.removeAllListeners('secureConnection')
  }

  // 4. **CORRECTION 1**: Explicitly terminate all tracked WebSocket clients
  //    Option A: Immediate termination (deterministic, no timeout)
  if (activeConnections.size > 0) {
    for (const ws of activeConnections) {
      ws.terminate() // Immediate termination without close frame
    }
    activeConnections.clear()
  }

  // 5. Close WebSocketServer (no longer accepting connections)
  if (wss) {
    wss.close()
    wss = null
  }

  // 6. Close HTTPS server (waits for TCP close, no active WS remain)
  if (server) {
    await new Promise<void>((resolve, reject) => {
      server!.close((err) => {
        if (err) reject(err)
        else resolve()
      })
    })
    server = null
  }

  // 7. Reset state (allow restart)
  boundAddress = null
  lifecycleState = 'STOPPED'
}
```

**Critical Shutdown Sequence** (CORRECTION 1):
1. **Mark SHUTTING_DOWN** → upgrade listener rejects new attempts immediately
2. **Remove listeners** → no new connections/upgrades processed
3. **Stop TCP listening** → no new TCP sockets accepted
4. **Terminate tracked clients** → `ws.terminate()` each active WebSocket (deterministic, immediate)
5. **Close WebSocketServer** → release ws library resources
6. **Close HTTPS server** → release Node TLS/TCP resources
7. **Nullify references** → enable garbage collection
8. **Mark STOPPED** → allow restart

**Shutdown Guarantees**:
1. New upgrade requests REJECTED during shutdown (lifecycleState check)
2. Active WebSocket connections TERMINATED explicitly (no reliance on ws.close() behavior)
3. HTTPS server closes AFTER all WebSocket clients terminated (clean shutdown)
4. **Restart Safety**: After `stop()` completes, `start()` creates entirely fresh instances (no socket/listener leaks)

**Double Stop Safety**: Idempotent for `STOPPED` state, throws error if called during `SHUTTING_DOWN`

**Active Socket Handling** (CORRECTION 1):
- `ws.terminate()` immediately destroys socket without sending close frame
- Deterministic: no timeout, no graceful close negotiation
- Rationale: B1a2a2 establishes transport only; graceful close is B1a2b/B1c concern

**Alternative Rejected** (Option B):
- Send close frame + timeout fallback → introduces timeout machinery
- Violates B1a2a2 scope (timeouts are B1a2b)
- Non-deterministic (timeout duration arbitrary)

### 3.4 Restart Behavior

**Scenario**: `stop()` followed by `start()`

**Guarantees** (CORRECTION 1):
1. New `server` instance created (old instance fully released)
2. New `wss` instance created (old instance released)
3. New `activeConnections` Set created (no client leak)
4. New `upgrade` listener bound (no duplication)
5. Lifecycle state transitions: RUNNING → SHUTTING_DOWN → STOPPED → RUNNING
6. **ZERO old sockets survive**: Explicit termination in stop() ensures clean slate

**Test Coverage Required**:
- Restart sequence (start → stop → start)
- Verify no duplicate upgrade events
- Verify new connections work after restart
- Verify old connections CANNOT survive restart (activeConnections.size === 0 after stop)

### 3.5 Upgrade During Shutdown

**Scenario**: Client sends upgrade request during `SHUTTING_DOWN` state

**Behavior**:
```typescript
server.on('upgrade', (req, socket, head) => {
  if (lifecycleState !== 'RUNNING' || !wss) {  // SHUTTING_DOWN or STOPPED
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

## PHASE 5 — TDD PROOF TABLE (CORRECTION 2: THREE-LEVEL TESTABILITY)

### 5.1 Transport Security Proof (Existing + New)

| Test Case | Policy A | mTLS | TLS 1.3 | Temporal | Upgrade | Expected Outcome |
|-----------|----------|------|---------|----------|---------|------------------|
| Valid fresh TLS 1.3 + mTLS + valid dates + valid upgrade | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | **WSS connection established** |
| Resumed TLS session | ❌ Fail | N/A | N/A | N/A | N/A | Socket destroyed (Policy A) |
| TLS 1.2 connection | N/A | N/A | ❌ Fail | N/A | N/A | Handshake rejected |
| Missing client certificate | N/A | ❌ Fail | N/A | N/A | N/A | Socket destroyed (mTLS) |
| Untrusted client certificate | N/A | ❌ Fail | N/A | N/A | N/A | Socket destroyed (mTLS) |
| Certificate not yet valid (real cert) | ✅ Pass | ✅ Pass | ✅ Pass | ❌ (TLS rejects) | N/A | **TLS handshake rejected OR socket destroyed** (see 5.2.C) |
| Certificate expired (real cert) | ✅ Pass | ✅ Pass | ✅ Pass | ❌ (TLS rejects) | N/A | **TLS handshake rejected OR socket destroyed** (see 5.2.C) |
| Certificate dates unparseable | ✅ Pass | ✅ Pass | ✅ Pass | ❌ CERT_DATES_UNPARSEABLE | N/A | **Socket destroyed BEFORE upgrade** |
| Certificate at exact notBefore | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | WSS connection established |
| Certificate at exact notAfter | ✅ Pass | ✅ Pass | ✅ Pass | ❌ CERT_EXPIRED | N/A | Socket destroyed BEFORE upgrade |
| Valid transport, invalid HTTP method (POST) | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Method | HTTP 400 + socket destroyed |
| Valid transport, malformed Upgrade header ("notwebsocket") | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Header | HTTP 400 + socket destroyed |
| Valid transport, missing Connection header | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Header | HTTP 400 + socket destroyed |
| Valid transport, invalid WebSocket version | ✅ Pass | ✅ Pass | ✅ Pass | ✅ Pass | ❌ Version | HTTP 400 + socket destroyed |

**Proof Metrics**:
- Upgrade count MUST be zero for all rejected connections
- Upgrade count MUST be exactly 1 for accepted connections
- Connection count MUST be exactly 1 for successful upgrades
- `activeConnections.size` MUST equal number of established connections

### 5.2 Certificate Temporal Validation Proof (CORRECTION 2: THREE-LEVEL TESTABILITY)

**Problem**: Production `rejectUnauthorized: true` means genuinely invalid certs MAY be rejected by TLS/OpenSSL BEFORE upgrade event. Cannot test "invalid cert reaches upgrade handler" with real TLS.

**Solution**: Three-level proof strategy

#### Level A: Pure Validator (Existing from B1a1)

**Purpose**: Prove core date comparison logic in isolation

| Test Case | notBefore | notAfter | now | Expected Result |
|-----------|-----------|----------|-----|-----------------|
| Valid range (mid-period) | 2024-01-01 | 2025-01-01 | 2024-06-15 | ✅ ok: true |
| Valid range (at notBefore) | 2024-06-15 12:00:00 | 2025-01-01 | 2024-06-15 12:00:00 | ✅ ok: true |
| Not yet valid (before notBefore) | 2025-01-01 | 2026-01-01 | 2024-06-15 | ❌ CERT_NOT_YET_VALID |
| Expired (after notAfter) | 2023-01-01 | 2024-01-01 | 2024-06-15 | ❌ CERT_EXPIRED |
| Expired (at notAfter) | 2023-01-01 | 2024-06-15 12:00:00 | 2024-06-15 12:00:00 | ❌ CERT_EXPIRED |

**Implementation**: Unit tests for `validateCertificateDates()` (PR-08B1a1, already exists)

#### Level B: Peer Certificate Adapter (New, Synthetic)

**Purpose**: Prove `validatePeerCertificateDates()` correctly extracts/parses PeerCertificate fields and delegates to Level A validator

**Test Strategy**: Use synthetic PeerCertificate-shaped objects (NOT real TLS handshakes)

```typescript
describe('validatePeerCertificateDates', () => {
  it('accepts valid date range from peer certificate', () => {
    const peerCert: Partial<tls.PeerCertificate> = {
      valid_from: 'Jan 1 00:00:00 2024 GMT',  // Node date format
      valid_to: 'Jan 1 00:00:00 2025 GMT',
      // ... other fields omitted for test focus
    }
    
    const result = validatePeerCertificateDates(peerCert as tls.PeerCertificate)
    
    expect(result.ok).toBe(true)
  })
  
  it('rejects certificate not yet valid', () => {
    const peerCert: Partial<tls.PeerCertificate> = {
      valid_from: 'Jan 1 00:00:00 2099 GMT',  // Future
      valid_to: 'Jan 1 00:00:00 2100 GMT',
    }
    
    const result = validatePeerCertificateDates(peerCert as tls.PeerCertificate)
    
    expect(result.ok).toBe(false)
    expect(result.error).toBe('CERT_NOT_YET_VALID')
  })
  
  it('rejects certificate expired', () => {
    const peerCert: Partial<tls.PeerCertificate> = {
      valid_from: 'Jan 1 00:00:00 2020 GMT',
      valid_to: 'Jan 1 00:00:00 2021 GMT',  // Past
    }
    
    const result = validatePeerCertificateDates(peerCert as tls.PeerCertificate)
    
    expect(result.ok).toBe(false)
    expect(result.error).toBe('CERT_EXPIRED')
  })
  
  it('rejects missing valid_from', () => {
    const peerCert: Partial<tls.PeerCertificate> = {
      valid_to: 'Jan 1 00:00:00 2025 GMT',
      // valid_from missing
    }
    
    const result = validatePeerCertificateDates(peerCert as tls.PeerCertificate)
    
    expect(result.ok).toBe(false)
    expect(result.error).toBe('CERT_DATES_UNPARSEABLE')
  })
  
  it('rejects missing valid_to', () => {
    const peerCert: Partial<tls.PeerCertificate> = {
      valid_from: 'Jan 1 00:00:00 2024 GMT',
      // valid_to missing
    }
    
    const result = validatePeerCertificateDates(peerCert as tls.PeerCertificate)
    
    expect(result.ok).toBe(false)
    expect(result.error).toBe('CERT_DATES_UNPARSEABLE')
  })
  
  it('rejects malformed date string', () => {
    const peerCert: Partial<tls.PeerCertificate> = {
      valid_from: 'invalid-date-format',
      valid_to: 'Jan 1 00:00:00 2025 GMT',
    }
    
    const result = validatePeerCertificateDates(peerCert as tls.PeerCertificate)
    
    expect(result.ok).toBe(false)
    expect(result.error).toBe('CERT_DATES_UNPARSEABLE')
  })
})
```

**Proof Coverage**:
- ✅ Valid date parsing (Node date format, NOT ISO 8601)
- ✅ Expired detection
- ✅ Not-yet-valid detection
- ✅ Missing field rejection
- ✅ Malformed field rejection
- ✅ Delegation to `validateCertificateDates()`

**NOT Real TLS Tests**: Explicitly synthetic PeerCertificate objects. Does NOT require real certificate generation or TLS handshakes.

#### Level C: Real TLS/WSS Integration (Valid Cert Only)

**Purpose**: Prove full stack with REAL TLS handshake, mTLS, Policy A, temporal validation, upgrade, and WSS connection

**Test Strategy**: Use REAL valid client certificate ONLY

```typescript
describe('WSS Integration with Real TLS', () => {
  it('accepts valid WSS upgrade with fresh TLS and valid certificate dates', async () => {
    // Arrange
    const gateway = createTlsGateway(config)
    await gateway.start()

    const clientCert = loadValidTestCertificate() // notBefore < now < notAfter

    // Act
    const client = await createTlsClient({
      cert: clientCert.cert,
      key: clientCert.key,
      ca: trustedCA,
      rejectUnauthorized: true  // Production setting
    })

    await client.upgradeToWebSocket('/') // Send GET with Upgrade headers

    // Assert
    expect(client.isWebSocketConnected()).toBe(true)
    expect(gateway.upgradeCount).toBe(1) // Proof: upgrade succeeded
    expect(gateway.activeConnections.size).toBe(1) // Proof: client tracked
    expect(gateway.productActionCount).toBe(0) // Proof: no business logic
  })
})
```

**Expired/Not-Yet-Valid Real Cert Test** (CORRECTION 2: Honest acknowledgment):

```typescript
describe('WSS Integration with Invalid Real TLS Certs', () => {
  it('rejects expired certificate (TLS or adapter)', async () => {
    // Arrange
    const gateway = createTlsGateway(config)
    await gateway.start()

    const expiredCert = loadExpiredTestCertificate() // notAfter < now

    // Act & Assert
    // IF OpenSSL rejects during TLS handshake:
    //   - TLS connection fails immediately
    //   - handleUpgrade never called
    //   - Proof: gateway.upgradeCount === 0, gateway.handleUpgradeCount === 0
    //
    // IF OpenSSL accepts (edge case: rejectUnauthorized doesn't check dates):
    //   - TLS succeeds, upgrade event fires
    //   - validatePeerCertificateDates rejects with CERT_EXPIRED
    //   - Socket destroyed before handleUpgrade
    //   - Proof: gateway.upgradeCount === 0, gateway.handleUpgradeCount === 0
    //
    // Either way: NO WebSocket connection established

    await expect(
      createTlsClient({
        cert: expiredCert.cert,
        key: expiredCert.key,
        ca: trustedCA,
        rejectUnauthorized: true
      })
    ).rejects.toThrow() // TLS handshake or socket destroyed

    expect(gateway.upgradeCount).toBe(0) // No WSS connection
    expect(gateway.handleUpgradeCount).toBe(0) // handleUpgrade not called
  })
  
  it('rejects not-yet-valid certificate (TLS or adapter)', async () => {
    // Same pattern as expired test above
    // Proof: gateway.upgradeCount === 0, gateway.handleUpgradeCount === 0
  })
})
```

**Honest Test Acknowledgment** (CORRECTION 2):
- Production `rejectUnauthorized: true` means OpenSSL MAY reject expired/not-yet-valid certs during TLS handshake
- Test CANNOT force "invalid cert reaches upgrade handler" with real TLS
- Test CAN prove: No WSS connection established (upgrade count = 0, handleUpgrade count = 0)
- Do NOT attribute TLS rejection to B1a1 adapter (honest about rejection boundary)
- Do NOT weaken `rejectUnauthorized` for tests (preserve production security posture)
- Do NOT add production test flags (no conditional logic in production code)

**Proof Metrics**:
- ✅ Full stack: TLS 1.3 → mTLS → Policy A → peer adapter → temporal OK → upgrade → WSS
- ✅ Invalid cert: TLS rejection OR adapter rejection → NO WSS connection (upgrade count = 0)
- ✅ Production security settings preserved (`rejectUnauthorized: true`)

### 5.3 Lifecycle Proof (CORRECTION 1: Deterministic Shutdown)

| Test Case | Expected Behavior | Proof Metric |
|-----------|-------------------|--------------|
| Start gateway | Server listens, WSS ready, state = RUNNING | No errors, address() returns host/port, lifecycleState === 'RUNNING' |
| Double start | Throws error | Error message: "Gateway already started or shutting down" |
| Stop gateway | Server closes, WSS closed, clients terminated, state = STOPPED | No errors, address() returns null, lifecycleState === 'STOPPED', activeConnections.size === 0 |
| Double stop | Idempotent (STOPPED), error (SHUTTING_DOWN) | STOPPED: no error; SHUTTING_DOWN: throws error |
| Restart (start → stop → start) | New server/WSS instances, no client leak | New connections accepted after restart, activeConnections.size === 0 after stop |
| Upgrade during shutdown | Socket destroyed, no WebSocket connection | lifecycleState === 'SHUTTING_DOWN' → upgrade rejected, upgrade count = 0 |
| Listener leak check (10 restarts) | No duplicate listeners | EventEmitter listener count stable |
| Active clients during stop() | All clients terminated explicitly | activeConnections.size === 0 after stop(), ws.terminate() called for each client |

**Shutdown Termination Proof** (CORRECTION 1):
```typescript
it('terminates all active WebSocket clients during stop', async () => {
  // Arrange
  const gateway = createTlsGateway(config)
  await gateway.start()
  
  const clients = await createMultipleTlsClients(5) // 5 valid WSS connections
  expect(gateway.activeConnections.size).toBe(5)
  
  // Act
  await gateway.stop()
  
  // Assert
  expect(gateway.activeConnections.size).toBe(0) // All clients removed
  expect(gateway.lifecycleState).toBe('STOPPED')
  
  // Verify clients disconnected
  for (const client of clients) {
    expect(client.isConnected()).toBe(false)
  }
})
```

### 5.4 Upgrade Header Validation Proof (CORRECTION 4: Token Matching)

| Test Case | Upgrade Header | Connection Header | Expected Result |
|-----------|----------------|-------------------|-----------------|
| Valid tokens | "websocket" | "Upgrade" | ✅ ok: true |
| Valid tokens (case-insensitive) | "WebSocket" | "upgrade" | ✅ ok: true |
| Valid tokens (multi-value) | "keep-alive, websocket" | "keep-alive, Upgrade" | ✅ ok: true |
| Invalid substring | "notwebsocket" | "Upgrade" | ❌ Header error (token mismatch) |
| Invalid substring | "websocket" | "notupgrade" | ❌ Header error (token mismatch) |
| Missing token | undefined | "Upgrade" | ❌ Header error |
| Missing token | "websocket" | undefined | ❌ Header error |

**Token Matching Unit Tests** (CORRECTION 4):
```typescript
describe('containsToken', () => {
  it('matches exact token (case-insensitive)', () => {
    expect(containsToken('websocket', 'websocket')).toBe(true)
    expect(containsToken('WebSocket', 'websocket')).toBe(true)
  })
  
  it('matches token in comma-separated list', () => {
    expect(containsToken('keep-alive, websocket', 'websocket')).toBe(true)
    expect(containsToken('websocket, keep-alive', 'websocket')).toBe(true)
  })
  
  it('rejects substring false positives', () => {
    expect(containsToken('notwebsocket', 'websocket')).toBe(false)
    expect(containsToken('websocketx', 'websocket')).toBe(false)
  })
  
  it('handles whitespace correctly', () => {
    expect(containsToken('  websocket  ', 'websocket')).toBe(true)
    expect(containsToken('keep-alive , websocket', 'websocket')).toBe(true)
  })
})
```

### 5.5 wsClientError Listener (CORRECTION 5)

**Decision**: **NOT required in B1a2a2**

**Rationale**:
- `ws` library default behavior: malformed handshakes send `HTTP 400` + close socket
- `validateUpgradeRequest()` rejects malformed requests BEFORE `handleUpgrade`
- Double rejection (our validator + ws internal) ensures no socket leak
- No evidence of socket leak in ws library for invalid handshakes

**Future Consideration**:
- IF socket leaks observed in production: Add `wss.on('wsClientError', (error, socket) => { socket.destroy() })`
- NOT preemptively added (YAGNI principle)

**Test Coverage**:
- Verify malformed upgrade requests rejected (validateUpgradeRequest tests)
- Verify upgrade count = 0 for malformed requests (no WebSocket created)

### 5.6 Scope Boundary Proof (What Doesn't Happen)

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

**Internal Changes** (CORRECTION 1):
- `start()`: Creates WebSocketServer, binds upgrade listener, initializes lifecycle state (INTERNAL)
- `stop()`: Transitions to SHUTTING_DOWN, terminates active clients, closes WebSocketServer, closes server (INTERNAL)

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

/**
 * Check if comma-separated header contains specific token (RFC 2616)
 * @param headerValue - Header value to parse
 * @param token - Token to search for (case-insensitive)
 * @returns True if token found, false otherwise
 * @internal Exported for testing only, not public API
 */
export function containsToken(headerValue: string, token: string): boolean
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
| `apps/desktop/electron/security/transport/tls-gateway.ts` | Modified | ~140 | Add WSS server, lifecycle state, client tracking, upgrade listener, validation functions, deterministic shutdown |
| `apps/desktop/package.json` | Modified | +2 | Add `ws` runtime + `@types/ws` dev dependencies |

**Total Production**: ~142 lines changed

### 7.2 Test Files

| File | Change Type | Lines Changed | Description |
|------|-------------|---------------|-------------|
| `apps/desktop/test/main/security/transport/tls-gateway.test.ts` | Modified | ~220 | Add WSS upgrade tests, temporal validation (3-level proof), lifecycle tests (deterministic shutdown), token matching tests |

**Total Tests**: ~220 lines changed

### 7.3 No New Files

All changes are extensions to existing files. No new modules introduced.

---

## PHASE 8 — SIZE ESTIMATE & DELIVERY GATE (CORRECTED)

### 8.1 Size Breakdown

| Category | Lines Changed | Notes |
|----------|---------------|-------|
| Production code | 142 | +30 from baseline (lifecycle state, client tracking, token matching, deterministic shutdown) |
| Test code | 220 | +20 from baseline (synthetic adapter tests, token matching tests, shutdown termination tests) |
| Dependencies | +2 | package.json (ws + @types/ws) |
| **Total** | **364** | **Within 400-line budget** ✅ |

**Delta from Original Estimate** (+52 lines):
- Lifecycle state management: +10 lines
- Client tracking Set + termination loop: +15 lines
- Token-based header validation (containsToken): +10 lines
- Enhanced test coverage (synthetic adapter + shutdown): +17 lines

**Gate Status**: ✅ **PASS** (≤400 lines)

### 8.2 Single PR Viability

**Recommendation**: Single PR delivery

**Rationale**:
- Size well within 400-line budget (364 total)
- Semantic coherence: WSS upgrade + temporal validation + lifecycle are tightly coupled
- No natural split point (lifecycle corrections integral to shutdown safety)
- Test coverage integrated (upgrade tests validate temporal validation + lifecycle)

**Risk Assessment**: Low
- No breaking changes (backward compatible)
- Isolated to transport layer (no business logic)
- Comprehensive test coverage prevents regressions
- Deterministic shutdown semantics (no timeouts, no races)

---

## PHASE 9 — RISK ANALYSIS & MITIGATION

### 9.1 Certificate Date Parsing Risk

**Risk**: `valid_from`/`valid_to` string format varies across Node versions or certificate types

**Mitigation**:
1. Defensive parsing with `new Date(string)` + `isNaN()` checks
2. Return `CERT_DATES_UNPARSEABLE` error (explicit failure mode)
3. Test coverage includes malformed date scenarios
4. Production logging (future) for unparseable dates to detect edge cases
5. Node 20 + Node 24 compatibility verified

**Date Format** (CORRECTION 3): **NOT ISO 8601**. Node PeerCertificate uses format like:
```
Aug 14 00:00:00 2017 GMT
```
`new Date()` constructor handles this format correctly in Node 20/24.

**Impact**: Low (Node.js TLSSocket API is stable, consistent format)

### 9.2 WebSocketServer Shutdown Race (RESOLVED)

**Risk**: Active WebSocket connection receives data between `wss.close()` and full shutdown

**Mitigation** (CORRECTION 1):
1. Explicit client tracking in `Set<WebSocket>`
2. `ws.terminate()` called for every tracked client during stop()
3. Lifecycle state prevents new upgrades during shutdown
4. Deterministic termination (no timeout, no race)

**Impact**: None (deterministic shutdown eliminates race)

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

### 9.5 Token Matching Edge Cases (NEW)

**Risk**: Malformed multi-value headers bypass token validation

**Mitigation** (CORRECTION 4):
1. Proper comma-splitting + trim + case-insensitive exact match
2. Unit tests cover: exact match, multi-value, whitespace, substring false positives
3. Aligns with RFC 2616 token semantics

**Impact**: Low (proper implementation, comprehensive tests)

---

## PHASE 10 — IMPLEMENTATION GUIDANCE

### 10.1 TDD Workflow

1. **Test First**: Write temporal validation tests (3-level proof: pure validator, synthetic adapter, real TLS)
2. **Red**: Tests fail (validation functions not implemented)
3. **Green**: Implement `validatePeerCertificateDates`, `validateUpgradeRequest`, `containsToken`
4. **Refactor**: Extract common error handling patterns
5. **Integration**: Add upgrade listener to gateway `start()`
6. **Lifecycle Tests**: Verify start/stop/restart semantics (deterministic shutdown)
7. **Scope Proof**: Verify product action count = 0

### 10.2 Certificate Test Data Generation

**Approach**: Use existing test certificate infrastructure (from Policy A tests)

**Test Certificates Required**:
- Valid certificate (notBefore < now < notAfter) — for Level C real TLS test
- Synthetic PeerCertificate objects (for Level B adapter tests) — no real cert generation needed

**Generation**: Extend existing certificate generation helpers (if needed for Level C only)

### 10.3 Integration Test Pattern (Level C)

**Scenario**: Real TLS 1.3 mTLS client performs WSS upgrade

```typescript
it('accepts valid WSS upgrade with fresh TLS and valid certificate dates', async () => {
  // Arrange
  const gateway = createTlsGateway(config)
  await gateway.start()

  const clientCert = loadValidTestCertificate() // notBefore < now < notAfter

  // Act
  const client = await createTlsClient({
    cert: clientCert.cert,
    key: clientCert.key,
    ca: trustedCA,
    rejectUnauthorized: true  // Production setting preserved
  })

  await client.upgradeToWebSocket('/') // Send GET with Upgrade headers

  // Assert
  expect(client.isWebSocketConnected()).toBe(true)
  expect(gateway.upgradeCount).toBe(1) // Proof: upgrade succeeded
  expect(gateway.activeConnections.size).toBe(1) // Proof: client tracked
  expect(gateway.productActionCount).toBe(0) // Proof: no business logic
})
```

### 10.4 Shutdown Test Pattern (CORRECTION 1)

```typescript
it('rejects upgrade during shutdown', async () => {
  // Arrange
  const gateway = createTlsGateway(config)
  await gateway.start()

  const client = await createTlsClient(validConfig)

  // Act
  const stopPromise = gateway.stop() // Begin shutdown (lifecycleState = SHUTTING_DOWN)
  const upgradePromise = client.upgradeToWebSocket('/') // Race upgrade

  // Assert
  await expect(upgradePromise).rejects.toThrow() // Socket destroyed
  await stopPromise // Clean shutdown completes
  expect(gateway.upgradeCount).toBe(0) // No upgrade succeeded
  expect(gateway.lifecycleState).toBe('STOPPED')
  expect(gateway.activeConnections.size).toBe(0) // All clients terminated
})

it('terminates active clients during stop', async () => {
  // Arrange
  const gateway = createTlsGateway(config)
  await gateway.start()
  
  const clients = await createMultipleTlsClients(5)
  expect(gateway.activeConnections.size).toBe(5)
  
  // Act
  await gateway.stop()
  
  // Assert
  expect(gateway.activeConnections.size).toBe(0) // Deterministic termination
  expect(gateway.lifecycleState).toBe('STOPPED')
  
  for (const client of clients) {
    expect(client.isConnected()).toBe(false)
  }
})
```

---

## PHASE 11 — EXPLICIT SCOPE BOUNDARIES (RECAP)

### 11.1 What B1a2a2 DOES Include

✅ WebSocketServer creation with `noServer: true` pattern  
✅ `httpsServer.on('upgrade', ...)` listener binding  
✅ Certificate temporal validation during upgrade (using `validateCertificateDates`)  
✅ Peer certificate extraction from TLSSocket (`getPeerCertificate()`)  
✅ Upgrade request validation (method, **token-based** headers, version)  
✅ Socket destruction on validation failures (BEFORE `handleUpgrade`)  
✅ Gateway lifecycle (start/stop) with **explicit client tracking + termination**  
✅ Lifecycle state machine (STOPPED → RUNNING → SHUTTING_DOWN)  
✅ Shutdown prevents new upgrades (lifecycle state check)  
✅ Deterministic shutdown (no timeouts, immediate `ws.terminate()`)  
✅ Restart safety (no listener leaks, no client leaks)  
✅ `ws` + `@types/ws` dependency addition  

### 11.2 What B1a2a2 DOES NOT Include

❌ X.509 business identity validation (installationId/stationId/centerId) → B1b  
❌ WebSocket message handling, parsing, or validation → B1c  
❌ Connection limits, max connections, concurrent connection enforcement → B1a2b  
❌ Payload size limits, message size validation → B1a2b  
❌ Connection timeouts, idle timeouts, ping/pong keepalive → B1a2b  
❌ Graceful close with timeout fallback → B1a2b  
❌ Rate limiting, request throttling → B1a2b  
❌ FSM composition, state machine integration → B1c  
❌ Product action dispatch, business logic → B1c  
❌ Session management, session store → B1c  
❌ Server certificate provisioning, certificate rotation → separate infrastructure  
❌ Clock plausibility validation, NTP synchronization → separate concern  
❌ Path-based routing, multi-endpoint support → future enhancement  
❌ Subprotocol negotiation (Sec-WebSocket-Protocol) → future enhancement  
❌ `wsClientError` listener → not required (ws default behavior sufficient)  

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
- [ ] Corrections ratified (lifecycle, testability, date format, token matching, wsClientError)
- [ ] Scope boundaries confirmed (no B1a2b/B1c absorption)
- [ ] Size estimate confirms ≤400 lines (364 actual)
- [ ] TDD proof table covers all security invariants (3-level temporal proof)
- [ ] Dependency justification accepted

### 13.2 Implementation Verification

- [ ] All TDD proof table tests implemented and passing (3-level temporal validation)
- [ ] Certificate temporal validation integrated BEFORE `handleUpgrade`
- [ ] Policy A rejection prevents upgrade (existing test still passes)
- [ ] TLS 1.2 / missing cert / untrusted cert rejected (existing tests still pass)
- [ ] Upgrade validation rejects malformed requests (token-based matching)
- [ ] Lifecycle tests (start/stop/restart) passing (deterministic shutdown)
- [ ] Listener leak test passing (10 restarts)
- [ ] Client termination test passing (activeConnections.size === 0 after stop)
- [ ] Product action count = 0 for all tests
- [ ] TypeScript compilation clean (no type errors)

### 13.3 Pre-Merge Verification

- [ ] All tests passing (new + existing)
- [ ] No regressions in Policy A behavior
- [ ] Code review completed
- [ ] Size within budget (≤400 lines actual: 364)
- [ ] Documentation updated (if public API changed)

---

## DESIGN AUTHORITY

This **corrected** design is authoritative for PR-08B1a2a2 implementation. Deviations require explicit design amendment with rationale.

**Correction Pass**: Ratified ONLY these 5 corrections:
1. WebSocket shutdown semantics (explicit client tracking + termination)
2. Temporal validation testability (3-level proof strategy)
3. Date format wording (Node format, NOT ISO 8601)
4. Upgrade header validation (token matching, NOT substring)
5. wsClientError listener (not required, documented rationale)

**Sign-off Required**: Architecture design corrections approved before implementation phase

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

**B1a2a2 Responsibility**: Validate client request (token-based matching), delegate handshake response to `ws` library

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

### ws.WebSocket Termination

**Graceful Close** (NOT in B1a2a2):
```typescript
ws.close(code, reason) // Sends close frame, waits for peer acknowledgment
```

**Immediate Termination** (B1a2a2 shutdown):
```typescript
ws.terminate() // Destroys socket immediately, no close frame
```

**Rationale**: Deterministic shutdown without timeout machinery (B1a2b concern)

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

## APPENDIX C — CORRECTION SUMMARY

| Correction | Problem | Solution | Impact |
|------------|---------|----------|--------|
| 1. Lifecycle | Incorrect shutdown semantics (wss.close() doesn't terminate clients) | Explicit client tracking + lifecycle states + deterministic ws.terminate() | +30 lines |
| 2. Testability | Cannot test expired cert reaching adapter with real TLS | 3-level proof: pure validator, synthetic adapter, real TLS (valid only) | +17 lines (tests) |
| 3. Date Format | Called PeerCertificate dates "ISO 8601" | Correct: Node date format ("Aug 14 00:00:00 2017 GMT") | Wording only |
| 4. Token Matching | Substring includes() matches false positives | containsToken() with proper comma/token semantics | +10 lines |
| 5. wsClientError | Unspecified necessity | Not required; ws default behavior + our validator sufficient | 0 lines (omitted) |

**Total Delta**: +52 lines (364 total, within 400-line budget)

---

## FINAL PRE-TDD ARCHITECTURE AUTHORITY

**Status**: APPROVED — READY FOR SPLIT IMPLEMENTATION

**Architect**: `sdd-design` — `anthropic/claude-sonnet-4-5` HIGH, Direct Anthropic — APPROVE

**Independent review**: `gentle-ai-verify` — `antigravity/gemini-3.1-pro` HIGH — APPROVE

This section supersedes every earlier conflicting size forecast, single-slice delivery recommendation, start transition, and shutdown ordering in this document.

### Final Size Accounting

The dependency cost was measured from a clean Dinamizador worktree based on `4cc81696b0b8f9de4868029f54b7abbb9197b9ae`:

| Area | Net lines |
| --- | ---: |
| Production/source forecast | 165 |
| Tests forecast | 247 |
| `apps/desktop/package.json` | 2 |
| `package-lock.json` | 33 |
| Dependencies total | 35 |
| **Combined unsplit forecast** | **447** |

The lockfile contains no unrelated churn. The previous 364/372-line forecasts omitted or undercounted final lifecycle work and dependency lockfile lines and are superseded.

The combined 447-line candidate lies in the 401–450 explicit-cohesion-exception range. No cohesion exception is requested or assumed. The previously proposed single-slice delivery is **SUPERSEDED** by the semantic split below.

### Delivery Status

**Parent PR-08B1a2a2**

- **Status**: CLOSED / INTEGRATED
- **Implemented**: PR-08B1a2a2a CLOSED / INTEGRATED; PR-08B1a2a2b CLOSED / INTEGRATED
- **Remaining**: none within PR-08B1a2a2

**Final parent capability**: secure WSS transport integration + safe gateway lifecycle.

**Non-claims**: this does NOT imply FSM ACTIVE, business identity authorization, resource hardening, or product actions.

#### PR-08B1a2a2a — WSS Validation Primitives & Dependencies

**Status**: CLOSED / INTEGRATED

**Integration record**:

- **Dinamizador commit**: `1e1a11ea5b6d2ff5209ca673a639dd1877605202`
- **Base**: `4cc81696b0b8f9de4868029f54b7abbb9197b9ae`
- **Actual size**: 179 net lines
- **Changed/churn**: 181 lines (180 additions + 1 deletion)
- **Files**: `apps/desktop/electron/security/transport/tls-gateway.ts`; `apps/desktop/test/main/security/transport/tls-gateway.test.ts`; `apps/desktop/package.json`; `package-lock.json`
- **Dependencies**: `ws` 8.22.0 (requested `^8.18.0`); `@types/ws` 8.18.2 (requested `^8.5.13`)
- **Tests**: TLS gateway 44 PASS; certificate-validator 5 PASS; SecurityAudit 13 PASS; secure-link 97 PASS; main/security 146 PASS; typecheck PASS
- **Independent implementation review**: `gentle-ai-verify`; `antigravity/gemini-3.1-pro`; HIGH; APPROVE
- **Native review**: four lenses complete (review-risk, review-resilience, review-readability, review-reliability); one repeated-header-values blocker corrected within the authorized review budget; final result APPROVED

Scope delivered:

- Add `ws` as a runtime dependency and `@types/ws` as a development dependency.
- Update `package-lock.json` without unrelated dependency churn.
- Add pure `containsToken()` validation with comma-separated, trimmed, case-insensitive exact-token semantics.
- Add pure `validateUpgradeRequest()` validation for `GET`, `websocket`, `upgrade`, and WebSocket version `13`.
- Keep `Sec-WebSocket-Key` validation explicitly delegated to `ws.handleUpgrade()`.
- Add `validatePeerCertificateDates(peerCert, now?)` as a defensive adapter over `validateCertificateDates()`.
- Reject missing or unparseable peer certificate dates without silently accepting them.
- Preserve the existing Level A certificate-date validator tests.
- Add Level B synthetic `PeerCertificate` adapter tests and header/token tests.

Safety boundary:

- No `WebSocketServer` instance.
- No HTTPS `upgrade` listener.
- No `handleUpgrade()` call.
- No accepted WebSocket client.
- No gateway lifecycle behavior change.
- No production TLS weakening or test-only production flag.

This slice is independently safe to merge because it introduces validation primitives and dependencies but no active WSS behavior.

#### PR-08B1a2a2b — WSS Integration & Safe Gateway Lifecycle

**Status**: CLOSED / INTEGRATED

**Order**: SECOND

**Forecast**: original forecast approximately 262 net lines; actual size 367 net lines

**Exact dependency**: PR-08B1a2a2a integrated

**Exact new Dinamizador base**: `1e1a11ea5b6d2ff5209ca673a639dd1877605202`

**Integration record**:

- **Base**: `1e1a11ea5b6d2ff5209ca673a639dd1877605202`
- **Implementation commits**: `780ab0435c87aada50d537bbd4847dca560f2de1` (feat: integrate secure WSS gateway lifecycle) and `eef27518862506a52f1f40dab5c870701d1be5d4` (fix: harden WSS startup rollback)
- **Final integrated Dinamizador HEAD (origin/master)**: `eef27518862506a52f1f40dab5c870701d1be5d4`, integrated by normal fast-forward, no force
- **Actual size**: 367 net lines — production 107, tests 260 (`tls-gateway.test.ts` 223 + `tls-gateway-startup.test.ts` 37), other 0
- **Changed-line churn**: 481 (informational only; the ADA size gate metric is NET)
- **Size policy**: PASS (<=400 net); no cohesion exception required
- **Files**: `apps/desktop/electron/security/transport/tls-gateway.ts`; `apps/desktop/test/main/security/transport/tls-gateway.test.ts`; `apps/desktop/test/main/security/transport/tls-gateway-startup.test.ts`
- **Unchanged**: `package.json`, `package-lock.json`, `certificate-validator.ts`, `security-audit.ts`, `contracts.ts`; `RejectionCode` remains 15; Policy A and `SSL_OP_NO_TICKET` preserved
- **Tests**: TLS gateway 53 PASS; startup hardening 1 PASS; certificate-validator 5 PASS; SecurityAudit 13 PASS; secure-link 97 PASS; main/security 156 PASS; typecheck PASS

Scope delivered:

- `WebSocketServer({ noServer: true })` and an HTTPS upgrade hook.
- Integrate the a2a validators and perform real WSS acceptance only after every transport check succeeds.
- Lifecycle state and failed-start rollback.
- Immediate shutdown acceptance barrier and active client tracking.
- Deterministic `ws.terminate()` and restart isolation.
- Real TLS/WSS integration tests.

The first active WebSocket acceptance lands atomically with the lifecycle guarantees below. No intermediate merge may accept WebSockets while retaining unsafe or incomplete shutdown behavior.

**Review disposition**:

- **Independent implementation review**: `gentle-ai-verify`; `antigravity/gemini-3.1-pro`; HIGH; APPROVE (final candidate eef2751, no findings).
- **Native review final label**: `ESCALATED — MAINTAINER OVERRIDE`. Native review is NOT approved, NOT passed, and NOT clean for a2b; the maintainer explicitly authorized integration despite the terminal native escalation, and the override does not erase the escalations.
- **Native lineage 1** `review-0e39319593ba6559` (candidate 780ab04): terminal `escalated` / `native_stop_required`. R3-001 REFUTED by the refuter. R3-002 (review-reliability, CRITICAL, inferential, unknown causality, unverified_location) was investigated and NOT REPRODUCED: ws 8.22.0 `handleUpgrade` -> `completeUpgrade` -> callback runs synchronously in one stack so `stop()` cannot interleave; `WebSocket.terminate()` destroys the socket and sends no close frame; ws installs socket error handling during the handshake; the cited source range did not match the described callback. No production change was made for R3-002.
- **Native lineage 2** `review-bcb44a14dbd148d7` (candidate eef2751): terminal `escalated` / `native_stop_required`. The risk, resilience, and readability lenses reported no findings. Blocking finding R3-001 (review-reliability, CRITICAL, inferential, unknown causality, unverified_location) claimed a concurrent `stop()` could observe lifecycle SHUTTING_DOWN while `shutdownPromise` is already null and throw. NOT REPRODUCED / contradicted by the actual ordering: `lifecycle = 'STOPPED'` is assigned before `shutdownPromise = null`, synchronously in the same block with no await between them, so another `stop()` cannot interleave; the reliability lens itself acknowledged this separately. No production change was made for this finding.
- **Override rationale**:
  - Findings were inferential.
  - Causal disposition remained unknown.
  - Claims were manually investigated and not reproducible.
  - Actual control flow contradicts the blocking scenarios.
  - Repeated reruns produced different CRITICAL inferential claims rather than a reproducible defect.
  - Further candidate mutation solely to obtain a clean automated result was not justified.
  - Both lineages are preserved unmodified as historical review evidence.

**Real defects found and fixed during implementation and review**:

1. **Unhandled WebSocket error**: an authenticated client sending a malformed frame could emit an unhandled `ws` error capable of crashing the Electron main process. Fix: the accepted-client error handler terminates the client safely. (commit 780ab04)
2. **Stop during in-flight start**: `stop()` during `start()` could let the gateway finish RUNNING afterwards. Fix: `pendingStart` coordination makes `stop()` wait for startup completion and then stop the resulting gateway. (commit 780ab04)
3. **WebSocketServer constructor / pendingStart order**: `pendingStart` was published before `WebSocketServer` construction, so a synchronous constructor failure could poison `pendingStart`, preventing retry and making `stop()` wait forever. Fix: construct `WebSocketServer` before publishing `pendingStart`. Regression test `tls-gateway-startup.test.ts` (isolated Vitest module mock, no production test seam). (commit eef2751)

**Scope confirmation**:

- **Delivered**: real TLS 1.3 WSS with mTLS and private CA; Policy A and `SSL_OP_NO_TICKET` preserved; temporal and Upgrade validators executed before `handleUpgrade`; second lifecycle check; active WebSocket ownership; termination on stop; failed-start rollback with retry; stop-during-start handled; restart isolation; malformed client frame cannot crash the process.
- **Absent**: B1a2b resource hardening; B1b identity mapping; B1c secure-link/FSM composition; FSM ACTIVE; product actions; Usuario PC changes.

### Next Roadmap Item

PR-08B1a2b transport/resource hardening is the NEXT roadmap item. Expected concerns remain connection limits, payload limits, idle timeout, heartbeat, rate limiting, and backpressure. It is NOT implemented here and, per the existing roadmap authority, requires a fresh preflight within the 400-line budget before any implementation gate. B1b remains separate and blocked on business identity architecture; B1c remains separate.

### Final Transport Ordering

```text
TLS 1.3
→ mTLS authorization
→ Policy A fresh-session enforcement
→ peer-certificate temporal adapter
→ HTTP Upgrade validation
→ ws.handleUpgrade()
→ WebSocket accepted
```

Secure-link negotiation, FSM `ACTIVE`, message handling, and product actions remain deferred.

### Failed-Start Contract

The gateway does not need a public `STARTING` state. Setup may accumulate in local temporary references while the externally observable lifecycle remains `STOPPED`.

`RUNNING` is assigned only after:

1. `listen()` succeeds; and
2. the bound address resolves successfully.

Any start failure must:

- reject `start()` with the original failure;
- remove partial listeners and close partial HTTPS/WSS resources on a best-effort basis;
- leave `lifecycleState = STOPPED`;
- leave `server = null`;
- leave `wss = null`;
- leave `boundAddress = null`;
- leave the owned active-WebSocket set empty;
- permit a subsequent valid `start()`.

Cleanup failure must never publish `RUNNING` or leave a partially eligible gateway.

Required TDD evidence includes occupied-port/listen failure, rejected start, null address and clean stopped state, no retained WSS/client resources, and a successful subsequent start without arbitrary-delay correctness checks.

### Final Shutdown Contract

`stop()` follows this order:

1. Set `lifecycleState = SHUTTING_DOWN` first.
2. Capture current HTTPS server and WSS references locally.
3. Initiate HTTPS `server.close()` immediately and retain its completion promise. This is the acceptance barrier for new TCP/HTTP connections.
4. Keep transport and security listeners installed for already in-flight sockets.
5. Reject every in-flight Upgrade attempt when lifecycle state is not `RUNNING`.
6. Call `ws.terminate()` for every tracked WebSocket and clear the owned set.
7. Close the `WebSocketServer`.
8. Await HTTPS server-close completion.
9. Remove/reset remaining listeners and references only after shutdown completion.
10. Set `server = null`, `wss = null`, `boundAddress = null`, keep the owned set empty, and finish at `STOPPED`.

Removing listeners is not used as the mechanism for stopping TCP acceptance. No graceful-close timeout, retry loop, heartbeat, or other B1a2b policy is introduced. If close reports an error, no new upgrade can become eligible, active WebSockets have already been terminated, and lifecycle state must never return to `RUNNING`.

Required TDD evidence includes immediate `SHUTTING_DOWN`, immediate acceptance stop, rejection of new WSS Upgrade attempts, termination of existing owned clients, zero surviving old clients when `stop()` resolves, and an isolated restart with exactly one fresh listener/connection path.

### Temporal Validation Evidence Classification

1. **Level A — pure validator**: Existing `validateCertificateDates()` tests prove before/exact `notBefore` and exact/after `notAfter` semantics.
2. **Level B — adapter tests**: Synthetic `PeerCertificate`-shaped inputs prove Node date-time string parsing, missing/malformed rejection, deterministic `now`, and delegated temporal outcomes. These are not real TLS handshake tests.
3. **Level C — real TLS/WSS integration**: A real valid client certificate proves the complete positive transport sequence. OpenSSL may reject a real expired or not-yet-valid certificate before HTTP Upgrade; in that case the valid claim is only zero WSS connections and zero `handleUpgrade()` calls.

Production retains `rejectUnauthorized: true`. No test flag or weakened TLS configuration is permitted.

### Preserved Scope Boundaries

The split does not absorb:

- PR-08B1a2b resource controls: connection limits, payload limits, idle timeouts, heartbeat, rate limiting, or backpressure;
- B1b business identity mapping for installation, station, or center identities;
- B1c secure-link composition, FSM activation, message processing, or product actions;
- clock plausibility validation; or
- physical certificate provisioning.

`RejectionCode` and wire rejection vocabulary remain unchanged. Product actions remain zero.

### Implementation Gate

PR-08B1a2a2a and PR-08B1a2a2b are CLOSED / INTEGRATED, and parent PR-08B1a2a2 is CLOSED / INTEGRATED. No further PR-08B1a2a2 implementation remains. PR-08B1a2b, B1b, and B1c are separate roadmap items.

---

**END OF CORRECTED ARCHITECTURE DESIGN**
