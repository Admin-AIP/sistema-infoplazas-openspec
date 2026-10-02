# PR-08B1a2b — Transport Resource Hardening & Liveness
## Production Architecture Design

**Change ID**: PR-08B1a2b
**Scope**: Connection limits, payload limits, heartbeat liveness
**Target**: ADA NOVA PLUS / Dinamizador Desktop
**Baselines**:
- Dinamizador master: eef27518862506a52f1f40dab5c870701d1be5d4
- PR-08B1a2a2 (WSS integration): CLOSED/INTEGRATED
- Clean worktree: C:/SisAIP/Soft_Dinamizador-pr-08b1a2a2b-wss-integration-lifecycle (HEAD eef2751)

**Authority Sources**:
- OpenSpec changes/PR-08B1a2a2/design.md (WSS transport baseline)
- OpenSpec changes/session-start-login-usuario-pc/pr-08-blocking-requirements-closure.md (roadmap)
- OpenSpec changes/session-start-login-usuario-pc/specs/dinamizador-usuario-pc-secure-link/spec.md (secure-link contract)
- Probed baseline facts: Node v24.18.0, ws 8.22.0, package.json allows ^8.18.0, lockfile pins 8.22.0

**Status**: COMPLETE / INTEGRATED

**Integrated Dinamizador commit**: `878a375a83ac8733ca77d38c45c1086e77acf7b3`
**Integrated tree**: `f83505c53240dcb1fcae083b10b8273974a404b2`
**Base commit**: `eef27518862506a52f1f40dab5c870701d1be5d4`
**Native review**: `review-55f3290fae878981`
**Review result**: APPROVED + ACKNOWLEDGED (authority BURNED)
**Size**: 344 NET / 364 CHURN
**Validation**: TypeScript PASS, main/security 170 PASS
**Product actions**: ZERO

**Review record**:

- **Architect**: `sdd-design`; `anthropic/claude-sonnet-4-5`; HIGH; verdict APPROVE (final pass, after revision passes requested by the parent gate).
- **Independent architecture review**: `gentle-ai-verify`; `antigravity/gemini-3.1-pro`; HIGH; verdict APPROVE (18-point checklist PASS, with evidence from the code baseline and the ws 8.22.0 source).
- **Parent gate**: the draft was returned CHANGES REQUIRED three times before approval. Defects resolved: an incoherent two-threshold liveness design, a false "detects idle" claim, a rate-limiting rationale that conflated peer replacement with rate limiting, module-level liveness state, a self-contradictory size forecast, a flaky pong-in-flight test design, unverified provenance claims and an unverified audit-sink claim.
- **Native Gentle review**: lineage `review-55f3290fae878981`; APPROVED + ACKNOWLEDGED; 4-lens capture group (risk/resilience/readability/reliability); provider refuter executed; 1 CRITICAL finding (R3-001) REFUTED by deterministic negative evidence; 6 informational findings (R3-002, R3-003, R3-004, R3-005, R3-006, R4-heartbeat-delay-overflow) classified WARNING/SUGGESTION, non-blocking.
- **Maintainer acceptance**: approving this design = acceptance of the decisions in Section 14.1 and of the residual risks in Section 15.1. The owner numeric values in Section 14.2 remain open and block production wiring only, not implementation or merge.

**Post-Integration Advisory Findings** (INFORMATIONAL / NON-BLOCKING):

The native review identified six informational findings that are NOT blockers and do NOT reopen PR-08B1a2b. These are recorded as FOLLOW-UP / ADVISORY work for future independent consideration:

- **R3-002** (WARNING): Informational transport hardening opportunity
- **R3-003** (WARNING): Informational transport hardening opportunity
- **R3-004** (WARNING): Informational transport hardening opportunity
- **R3-005** (WARNING): Informational transport hardening opportunity
- **R3-006** (WARNING): Informational transport hardening opportunity
- **R4-heartbeat-delay-overflow** (WARNING): Deterministically reproduced numeric overflow risk in heartbeat delay calculation (non-blocking; classified informational by native review)

These findings remain informational. They do not invalidate the approved candidate, do not block B1b or B1c, and are not prerequisites for production deployment. Future independent hardening work may address them as separate bounded slices.

---

## 1. EXECUTIVE DECISION TABLE

This table is the canonical resource control specification. Each row defines ONE control with exact threshold authority and implementation status.

| Control | Exact Resource/Mechanism | Threshold Source | Known Value | Config-Required Value (blocks production wiring only) | Deferred Portion |
|---------|-------------------------|------------------|-------------|-------------------------------------------------------|------------------|
| **CONNECTION LIMIT** | Raw TCP sockets (pre-TLS, pre-upgrade); `https.Server.maxConnections` counts ALL sockets | **C** (operational policy; no business bound exists) | NO (requires owner decision) | NO for implementation; CONFIG REQUIRED value blocks PRODUCTION WIRING only: maxConnections (integer) | NONE (fully implemented in B1a2b) |
| **PAYLOAD** | WebSocket message payload bytes; `ws.maxPayload` enforces at parse time | **C** (operational policy; no spec bound exists) | NO (requires owner decision) | NO for implementation; CONFIG REQUIRED value blocks PRODUCTION WIRING only: maxPayload (integer bytes) | NONE (fully implemented in B1a2b) |
| **IDLE TIMEOUT** | Transport-observable "no inbound frames" is defeated by heartbeat pongs; application idleness (no secure-link messages) is B1c concern | Roadmap lists separately, but no independent transport control exists | N/A (absorbed into heartbeat) | N/A (absorbed into heartbeat; app-level timeout is B1c dependency) | ABSORBED into heartbeat dead-peer detection; residual risk: app-level idle (upgraded-but-never-negotiates) deferred to B1c handshake timeout |
| **HEARTBEAT** | Server-initiated WebSocket ping → client pong; awaiting-pong pattern detects DEAD peers (not idle) | **C** (operational policy; liveness probe frequency is deployment-specific) | NO (requires owner decision) | NO for implementation; CONFIG REQUIRED value blocks PRODUCTION WIRING only: pingIntervalMs (integer milliseconds) | NONE (fully implemented in B1a2b) |
| **RATE LIMITING** | Pre-upgrade connection/handshake attempt RATE keyed by remote address (IP) | N/A (no authoritative requirement, no threshold/window, no product decision on LAN abuse model) | NO (blocked on product decision: what rate/window, what key, whether warranted) | YES: rate threshold, time window, key trust assumptions, whether the LAN abuse model warrants it | FULLY DEFERRED to unscheduled independent follow-up slice (see Section 5); NOT blocked on B1c; requires explicit owner decision |
| **BACKPRESSURE** | Inbound: ws parser resource limits (maxPayload + ws defaults). Outbound: bufferedAmount monitoring (requires application messages) | Inbound: **B** (ws defaults) + **C** (maxPayload). Outbound: **C** (B1c concern) | Inbound YES (ws defaults + maxPayload config). Outbound NO (B1c) | NO for inbound (defaults + config adequate); outbound N/A (no messages in B1a2b) | Outbound backpressure deferred to B1c (no application messages to queue in B1a2b) |

**Threshold Classification**:
- **A** (authoritative): Defined by spec or protocol standard (NONE in B1a2b)
- **B** (technical invariant): Library/runtime defaults or documented safety bounds (ws/Node.js defaults)
- **C** (operational policy): Deployment-specific security/capacity trade-offs requiring owner decision (maxConnections, maxPayload, pingIntervalMs)

---

## 2. THRESHOLD AUTHORITY TABLE

Every numeric value is classified; every **C** threshold is a REQUIRED config field with NO production default.

| Threshold | Category | Authority/Justification | Value Source | Configured in B1a2b | Version-Dependent Risk |
|-----------|----------|------------------------|--------------|---------------------|------------------------|
| `maxConnections` | **C** | No business requirement bounds concurrent stations; security/capacity trade-off is deployment-specific | REQUIRED config field; synchronous throw if missing/invalid | YES | NO |
| `maxPayload` (WS message bytes) | **C** | No spec defines maximum secure-link message size; operational DoS threshold | REQUIRED config field; synchronous throw if missing/invalid | YES | NO |
| `pingIntervalMs` | **C** | No spec defines liveness probe frequency; network/responsiveness trade-off | REQUIRED config field; synchronous throw if missing/invalid | YES | NO |
| `perMessageDeflate` | **B** (security invariant) | MUST be `false` to prevent compression side-channel attacks (CRIME/BREACH-class) | **Explicit** in WebSocketServer options (hardcoded false, not relying on ws default) | YES (explicit false) | NO (explicit guarantees immutability) |
| `maxBufferedChunks` | **B** | ws 8.22.0 default 262144 (probed); prevents unbounded buffering during parse; safe ceiling | ws library default (NOT configured in B1a2b) | NO | YES (ws upgrade may change default) |
| `maxFragments` | **B** | ws 8.22.0 default 16384 (probed); prevents fragmentation DoS; safe ceiling | ws library default (NOT configured in B1a2b) | NO | YES (ws upgrade may change default) |
| Node.js `handshakeTimeout` | **B** | 120000 ms (2 minutes) documented default; mitigates half-open TLS handshakes exhausting connection budget | Node.js tls documented default (NOT probed, NOT configured) | NO | YES (Node.js upgrade may change) |
| Node.js `headersTimeout` | **B** | 60000 ms (probed on net.Server; https.Server inherits); mitigates slow HTTP header attacks | Node.js https default (NOT configured) | NO | YES (Node.js upgrade may change) |
| Node.js `requestTimeout` | **B** | 300000 ms (probed on net.Server; https.Server inherits); bounds HTTP request lifetime | Node.js https default (NOT configured) | NO | YES (Node.js upgrade may change) |

**Version-Dependent Risk**: Reliance on ws defaults (`maxBufferedChunks`, `maxFragments`) and Node.js defaults (handshake/header/request timeouts) is implicit and version-dependent. This is an accepted risk with a revisit trigger on ws/Node.js upgrades. Recommendation: pin ws to exact version or configure explicit values in a future hardening slice. Package.json allows ^8.18.0 while lockfile pins 8.22.0; a ws library upgrade could silently change defaults.

---

## 3. PROBED BASELINE FACTS (Node v24.18.0 / ws 8.22.0)

**Node.js https.Server defaults** (probed):
- `maxConnections`: `undefined` (no limit unless explicitly set)
- `headersTimeout`: 60000 ms
- `requestTimeout`: 300000 ms
- `keepAliveTimeout`: 5000 ms
- `timeout`: 0 (socket idle timeout disabled)
- `maxHeaderSize`: 16384 bytes
- `connectionsCheckingInterval`: 30000 ms

**Node.js tls defaults** (documented, NOT probed):
- `handshakeTimeout`: 120000 ms (2 minutes)

**ws WebSocketServer defaults** (probed from ws 8.22.0 source):
- `maxPayload`: 104857600 bytes (100 MiB)
- `maxBufferedChunks`: 262144
- `maxFragments`: 16384
- `perMessageDeflate`: `false` (default; B1a2b makes this EXPLICIT to prevent library default change)
- `autoPong`: `true` (server auto-responds to client-initiated pings)
- `closeTimeout`: 30000 ms
- `clientTracking`: `true`

**Probed Behavior Facts**:
- `net.Server.maxConnections = 2` closes the 3rd TCP connection and emits `'drop'` event
- ws `maxPayload` exceeded: ws sends close 1009 (CLOSE_MESSAGE_TOO_BIG), then emits server `'error'` event with `WS_ERR_UNSUPPORTED_MESSAGE_LENGTH`; with terminate-on-error handler, client still observes 1009
- `ws.ping()` behavior on non-OPEN socket (verified from ws 8.22.0 source, websocket.js lines 380-399):
  - If `readyState === CONNECTING`: throws synchronously
  - If `readyState === CLOSING` or `CLOSED`: calls `sendAfterClose` which invokes callback with error on next tick (does NOT throw)
- ws client option `autoPong` exists (websocket.js lines 650/683/705)

**Authoritative Absences** (verified from spec/roadmap/tasks):
- NO numeric authority for connection cap, payload limit, ping interval, rate limiting
- Roadmap lists "idle timeout" separately, but transport-observable idle is defeated by heartbeat pongs (see Section 4.1)
- "At most one ACTIVE link per station_id" is peer-REPLACEMENT semantics (B1b/B1c), NOT a rate limit
- `HANDSHAKE_TIMEOUT` is a B1c negotiation wire code, NOT a transport-level numeric threshold
- `TRANSPORT_REJECTION_REASONS` is closed to `['TLS_SESSION_RESUMED']`; no resource events audited in B1a2b

**Cross-Repo Conformance Dependency** (assumption to verify on Usuario PC side):
- The Usuario PC WebSocket client MUST maintain an active read loop to answer server pings (standard WebSocket client behavior with auto-pong enabled by default). If the client disables auto-pong or does not read from the socket, heartbeat liveness will falsely detect it as a dead peer. Verification deferred to cross-repo integration testing.

---

## 4. CONNECTION LIMIT DESIGN

### 4.1 Resource Definition & Enforcement

**Exact Resource**: Raw TCP sockets accepted by `https.Server`, including pre-TLS handshake sockets, sockets undergoing TLS handshake, post-TLS sockets awaiting HTTP Upgrade, and accepted WSS connections.

**Why Raw Socket Cap**: A limit applied only to accepted WSS clients would allow an attacker to exhaust server resources by opening TCP connections or performing TLS handshakes without completing upgrade. Native `maxConnections` prevents this at the earliest stage.

**Enforcement Mechanism**: `https.Server.maxConnections` property (Node.js native, race-safe)

**Enforcement Point**: Set `nextServer.maxConnections = config.maxConnections` after `https.createServer()` in `start()`

**Rejection Behavior** (probed on net.Server; https.Server behaviour to be proven by the real-TLS test):
- When `maxConnections` is set and current socket count equals the limit, additional TCP connections are closed by the server
- Server emits `'drop'` event (native; NOT handled in B1a2b)
- Exact mechanism (TCP RST vs FIN, whether TLS handshake occurs) to be proven by real-TLS test
- NO application-level rejection handler needed
- NO wire code (rejection is pre-application, pre-TLS, pre-identity)
- NO audit event (transport-layer rejection with no peer identity; SecurityAudit vocabulary unchanged)

**Accounting**: Automatic (Node.js manages socket count internally); increment on accept, decrement on socket `'close'`

**Restart Behavior**: `maxConnections` re-applies on gateway restart; no persistent state

### 4.2 Starvation Risk & Residual Risks

**Half-Open Socket Starvation**: Raw socket cap is shared by half-open sockets (TLS handshake in progress) and fully established WSS clients. An attacker could exhaust the connection budget with slow TLS handshakes, starving legitimate clients.

**Mitigations**:
1. **Native TLS handshakeTimeout** (Node.js documented default: 120000 ms, NOT probed): Bounds time a socket can remain in handshake, preventing indefinite resource hold
2. **Native headersTimeout** (Node.js default: 60000 ms, probed): Bounds HTTP header parsing time after TLS, preventing slow-header attacks
3. **Policy A + mTLS**: Only clients with valid certificates can complete TLS; unauthorized clients destroyed early

**Residual Risks**:
1. **Reconnect Overlap (Silent Drop)**: If a client's old socket is SILENTLY dropped (old socket still open server-side, e.g., network partition, no clean close), the client opening a new connection may hold two slots for up to 2x `pingIntervalMs` (old socket held until liveness timeout; see Section 6). This does NOT apply to a clean close (clean close immediately releases the slot). The operator MUST size `maxConnections` with margin for expected silent-drop reconnect overlap.
2. **Half-Open Starvation**: Half-open sockets can consume connection slots until native timeouts expire (up to 2 minutes for `handshakeTimeout`). A concurrent-connection cap bounds concurrency, NOT the RATE of connection attempts/TLS handshakes (CPU churn from repeated handshake attempts is unmitigated; see Section 5.4).
3. **Authenticated-Slot Hoarding**: An attacker with valid mTLS credentials could open `maxConnections` sockets, answer pings (stay alive), and never negotiate (hoard connection slots until B1c's handshake timeout is implemented). Heartbeat detects DEAD peers only, not idle peers. Mitigation requires B1c's application-level handshake timeout (currently an OPEN dependency).

**Assessment**: Native timeouts provide adequate bounded resource occupation. A second accepted-WSS cap is NOT needed in B1a2b; if operational experience reveals starvation, it can be added in a follow-up.

### 4.3 Configuration & Test Strategy

**Config Field**: `maxConnections: number` (REQUIRED, validated as finite positive safe integer at construction)

**Test Approach** (integration-level, real TLS/WSS):
- Create gateway with `maxConnections: 2`, start gateway
- Open 2 real mTLS WSS clients (both reach `'open'` event)
- Attempt 3rd connection: client does NOT reach `'open'` (ends in `'error'` or `'close'` event; event-driven settlement, NO timeout-based pass)
- Close one client, then open a new client (proves slot release, new client reaches `'open'`)
- Test home: New describe block `'Transport Resource Controls (b1a2b)'` in tls-gateway.test.ts

**Shared Test Limits**: Test `maxConnections` value (e.g., 10) must be generous enough that NO existing test trips the limit. Existing tests in `'WSS gateway integration lifecycle (a2b)'` open at most 2 simultaneous WSS clients per test. The connection cap boundary test uses its own small value (2).

---

## 5. PAYLOAD LIMIT DESIGN

### 5.1 Resource Definition & Enforcement

**Exact Resource**: WebSocket message payload size in bytes; applies to individual WS message payloads (text or binary frames) and reassembled fragmented messages. Does NOT parse application/business payload structure (B1c concern).

**Enforcement Mechanism**: `WebSocketServer({ maxPayload })` option (ws library native)

**Enforcement Point**: In `start()`, construct WebSocketServer with:
```typescript
new WebSocketServer({
  noServer: true,
  maxPayload: config.maxPayload,
  perMessageDeflate: false,  // Explicit: prevent compression side-channels
})
```

**Rejection Behavior** (probed on ws 8.22.0):
1. When message exceeds `maxPayload`, ws sends close frame with code **1009** (CLOSE_MESSAGE_TOO_BIG, standard WebSocket close code) to the client
2. ws emits `'error'` event on the server-side WebSocket with `WS_ERR_UNSUPPORTED_MESSAGE_LENGTH`
3. Existing client error handler (`client.on('error', () => client.terminate())`) terminates the socket
4. Client observes close code 1009; server observes 1006 (abnormal closure) due to terminate

**Related Controls** (ws 8.22.0 defaults, NOT configured in B1a2b):
- `maxBufferedChunks` (default 262144, probed): Prevents unbounded buffering during message reassembly
- `maxFragments` (default 16384, probed): Prevents fragmentation DoS by limiting message fragment count
- Both are **B-category technical defaults** (ws library); version-dependent risk accepted with revisit trigger on ws upgrade

**Compression Decision**: `perMessageDeflate` MUST be set **explicitly to false** in `WebSocketServer` options. This prevents compression side-channel attacks (CRIME/BREACH-class) and ensures a library default change cannot silently enable compression. This is a **B-category security invariant** (one line, explicit hardcoded value).

### 5.2 Fail-Closed Behavior

**Detection**: ws library (native)
**Enforcement**: Close 1009 sent, then terminate
**Cleanup**: Client removed from `activeClients` Set on `'close'` event
**Wire Code**: 1009 (standard WebSocket close code, NOT a secure-link RejectionCode)
**Audit**: NO (transport-level enforcement; no identity; SecurityAudit contract unchanged)

### 5.3 Configuration & Test Strategy

**Config Field**: `maxPayload: number` (REQUIRED, bytes, validated as finite positive safe integer at construction)

**Test Approach** (integration-level, real TLS/WSS):
- Create gateway with `maxPayload: 16`, start gateway
- Open real mTLS WSS client, send 17-byte message from client
- Client observes close event with code 1009 (behavioral proof)
- After payload violation, open a new client and prove it reaches `'open'` (server stayed healthy)
- Test home: `'Transport Resource Controls (b1a2b)'` describe block

---

## 6. HEARTBEAT LIVENESS DESIGN (IDLE TIMEOUT ABSORBED)

### 6.1 Design Rationale: Why Idle Timeout is Absorbed/Deferred

**Roadmap Wording**: The roadmap lists *idle timeout* and *heartbeat* as separate controls.

**Analysis**:
- **Heartbeat** (ping/pong) detects DEAD/unresponsive peers: a client that does not answer a ping is terminated.
- **Transport-observable idle**: A "no inbound frames" timeout would detect when a client sends NO frames of any kind. BUT: if the server sends pings, a live-but-quiet client sends pongs, which are inbound frames. Therefore, a transport-level "idle timeout" based on frame activity CANNOT distinguish a healthy quiet station from a truly idle one—the pongs defeat it.
- **Application-level idle**: A secure-link session that never sends a hello/action message is idle at the APPLICATION layer, not the transport layer. This is the "upgraded but never negotiates" case, which maps to the spec's HANDSHAKE_TIMEOUT concept (a B1c negotiation/state-machine timeout).

**DECISION**: "Idle timeout" as a distinct transport control is **ABSORBED/DEFERRED**:
- **Absorbed**: The heartbeat mechanism detects dead peers, which is the defensible transport-level concern.
- **Deferred**: Application-level idleness (no secure-link protocol messages) is deferred to B1c as the handshake timeout (the negotiation layer MUST enforce "if no hello arrives within X seconds of upgrade, terminate").

**Deviation from Roadmap**: This is a deviation from the roadmap wording that lists idle timeout separately. The reason: transport-observable idleness cannot be distinguished from a healthy quiet station when heartbeat is active. The product requirement (detect stuck/stale sessions) maps to B1c's handshake timeout, not a second transport timer.

**Residual Risk**: An mTLS-authenticated peer that completes the upgrade and answers pings but never speaks (no hello) keeps occupying a connection slot until B1c delivers the handshake timeout. **B1c DEPENDENCY**: B1c MUST implement a handshake timeout to close this gap. B1c owns that timeout and its value; B1a2b emits NO wire code for liveness/idle termination. Until then, a malicious/broken authenticated peer can hoard connection slots indefinitely by answering pings but never negotiating.

**Maintainer Acceptance Required**: Approving this design = acceptance of the idle timeout deviation and the residual authenticated-slot-hoarding risk until B1c delivers its handshake timeout.

### 6.2 Liveness Mechanism: Awaiting-Pong Pattern

**"Liveness" Definition** (transport-observable only):
- The client responds to server-initiated pings within the ping interval
- Does NOT detect application-layer idleness (no secure-link messages)
- Does NOT depend on business-layer heartbeat messages (autonomous operation concern, distinct from transport liveness)

**Operational Threshold**: ONE threshold only: `pingIntervalMs` (how often to ping all clients). Dead-peer detection latency is between 1x and 2x `pingIntervalMs` (deterministic).

**Why NOT Two Thresholds** (pingIntervalMs + pongTimeoutMs): The awaiting-pong pattern uses ONE operational threshold and simplifies: mark clients awaiting-pong after ping, terminate if still marked at next tick. No per-client timestamps, no `Date.now()`, no extra timers, no `WeakMap`.

**State Ownership** (per-gateway closure, NOT module-level):
- ALL liveness state (the interval handle, the awaiting-pong Set) is owned by the `createTlsGateway` closure and created FRESH per `start()` generation
- The interval callback closure captures the closure-scoped state (lifecycle, activeClients, awaiting-pong Set)
- A stale interval callback from a previous `start()` generation CANNOT act on a new generation's clients (fresh closure per start)
- This ensures restart isolation: each gateway's restart creates a fresh generation with independent liveness state

**Exact Generation Mechanism**: Each call to `start()` creates a LOCAL generation object (its own awaiting-pong Set and interval handle variable), and the interval callback closes over THAT local generation object (plus the shared activeClients/lifecycle references). `stop()` clears that generation's interval handle. The gateway keeps only the reference needed by `stop()`. Pseudo-structure:
```
function start() {
  const generation = { awaitingPong: new Set<WebSocket>(), intervalHandle: null };
  generation.intervalHandle = setInterval(() => { /* tick uses generation.awaitingPong */ }, ms);
  currentGeneration = generation; // gateway keeps reference for stop()
}
```

**Data Structure**: A `Set<WebSocket>` (closure-scoped, NOT module-level, NOT WeakSet) tracks clients currently awaiting a pong (they have been pinged but have not yet ponged). Cleared and recreated on each start.

### 6.3 Server-Initiated Ping Cycle

**Ping Loop**:
- The gateway's single `setInterval(pingIntervalMs)` starts when lifecycle = RUNNING (after successful bind)
- Created in the SAME synchronous step as `lifecycle = 'RUNNING'` as the last statement of the `start()` try block (so no failure path exists after it and a failed start never creates one)
- On each tick (guarded by try/catch to prevent main process crash):
  1. Check `lifecycle !== 'RUNNING'` → return immediately (guard against stop during tick)
  2. Iterate the awaiting-pong Set: any client still present (missed the previous ping) is `terminate()`d
  3. Clear the awaiting-pong Set
  4. Iterate `activeClients` Set: for each client with `readyState === WebSocket.OPEN`, add to awaiting-pong Set and call `ws.ping()`
  5. Clients with `readyState !== OPEN` are skipped (to avoid exceptions from `ws.ping()` on CONNECTING/CLOSING/CLOSED sockets)

**ws.ping() Behavior on Non-OPEN Socket** (verified from ws 8.22.0 source, websocket.js lines 380-399):
- If `readyState === CONNECTING`: throws synchronously
- If `readyState === CLOSING` or `CLOSED`: calls `sendAfterClose` which invokes callback with error on next tick (does NOT throw)
- Design implication: The ping loop MUST skip clients with `readyState !== OPEN` OR wrap ping call in try/catch

**Pong Handling**:
- `client.on('pong')` listener attached when client added to `activeClients`
- On pong: remove client from awaiting-pong Set (client proven alive)

**No Auto-Pong Interference**:
- ws server default `autoPong: true` (probed default) responds to **client-initiated** pings automatically
- Server-initiated `ws.ping()` requires explicit client pong response; standard ws client libraries auto-pong by default (conformance dependency on Usuario PC client: see Section 3)

### 6.4 Lifecycle Integration

**Start**:
- Create the gateway's single `setInterval` timer AFTER `lifecycle = 'RUNNING'` (after successful bind)
- Timer fires after the first `pingIntervalMs` delay, then every `pingIntervalMs`
- Create a fresh awaiting-pong Set (empty, closure-scoped)
- Interval created as the LAST statement of the try block in `start()` (no failure path after it)

**Client Addition** (on `'connection'` event):
- Add client to `activeClients` Set
- Attach `client.on('pong', () => awaitingPongSet.delete(client))`
- NO per-client timer (the gateway's single interval manages all clients)

**Client Removal (Single Owner)**:
- The existing `client.on('close')` handler is the SINGLE place that removes clients from `activeClients`; it also removes from awaiting-pong Set if present
- The ping tick does NOT remove from `activeClients`; it only calls `client.terminate()` on unresponsive clients
- The `'close'` event always fires after `terminate()`, so the `'close'` handler cleans up
- This keeps `stop()`'s terminate-all iteration correct (no concurrent modification of `activeClients` during iteration)

**Stop**:
- Set `lifecycle = 'SHUTTING_DOWN'`
- Call `server.close()` (existing acceptance barrier)
- `clearInterval(pingLoopInterval)` FIRST (before terminating clients, to ensure no ping attempts on clients being destroyed)
- Terminate all `activeClients` (existing behavior)
- Awaiting-pong Set cleared (or left to GC)

**Restart**:
- Stop clears interval and clients
- Start creates new interval with fresh awaiting-pong Set
- NO timer/listener leaks across restart: a fresh interval and fresh bookkeeping per start; nothing from the previous generation can ping/terminate new clients

**Failed-Start Rollback**: The interval is created in the same synchronous step as `lifecycle = 'RUNNING'` as the last statement of the try block. No failure path exists after the interval is created. If listen or wss construction fails, the interval is never created. The `pendingStart` coordination guarantees `stop()` waits for an in-flight start, so no race exists.

### 6.5 Race Conditions & Edge Cases

**Race: Stop During Ping**: Ping loop checks `lifecycle === 'RUNNING'` at the TOP of each tick; if lifecycle = SHUTTING_DOWN, loop exits immediately (no ping/terminate logic, no crash).

**Race: Pong Arrives After Timeout Decision**: If a client is terminated due to missing pong and a late pong arrives, the pong handler calls `awaitingPongSet.delete(client)` (benign; client already removed from `activeClients` and terminated). `terminate()` is idempotent per ws library.

**Race: Client Close During Ping Send**: `ws.ping()` on CLOSING/CLOSED socket does NOT throw (verified from ws source: calls `sendAfterClose` which invokes callback with error). Design: skip clients with `readyState !== OPEN` OR wrap in try/catch.

**Edge: First Ping Before Client Sends Anything**: Client may receive ping before sending any application message (valid); standard WebSocket client auto-pongs → clears awaiting-pong mark → liveness proven.

### 6.6 Configuration & Test Strategy

**Config Field**: `pingIntervalMs: number` (REQUIRED, validated as finite positive safe integer at construction)

**Test Approach** (fake timers for policy, real TLS/WSS for boundaries):
- **Fake Timers**: `vi.useFakeTimers({ toFake: ['setInterval', 'clearInterval'] })` installed BEFORE gateway start so the gateway's interval is captured; TLS/ws/net internal timers and Date stay real
- **Responsive Client Test with Ordering Barrier**: Start gateway, connect real mTLS WSS client (standard auto-pong), advance time by `pingIntervalMs` with `vi.advanceTimersByTime()`, await real `'ping'` event on client. THEN enforce deterministic ordering: the client sends `client.ping()` and awaits its own `'pong'` event (the server's autoPong only replies after the server has processed the client's earlier pong frame from the server's ping, because frames on one TLS socket are processed in order). Only after this barrier, advance the next tick. Verify client NOT terminated over several ticks. This barrier prevents a pong-in-flight race where the test advances time before the server processes the auto-pong, avoiding flaky termination of a healthy client. NO sleeps used; ordering enforced by frame-processing guarantees
- **Unresponsive Client Test**: Start gateway, connect real mTLS WSS client created with `autoPong: false` (ws 8.22.0 client option verified from ws source: websocket.js lines 650/683/705), advance time by 2x `pingIntervalMs`, await real `'close'` event on client (proves dead-peer termination)
- **Interval Cleanup Observable**: With fake timers installed, `vi.getTimerCount()` returns the number of pending fake timers; assert exactly 1 while RUNNING, 0 after `stop()`, 0 after a failed `start()` (occupied-port start), and exactly 1 (not 2) after `stop()` + `start()` (proves restart isolation and no timer leak)
- **Restart Isolation**: Start gateway, connect client A, `stop()`, `start()` again, connect client B, advance time by exactly one `pingIntervalMs`, observe client B receives exactly one ping (proves new interval independent of old generation; client A cannot be observed after stop because it was already terminated)
- **FALLBACK if Narrow Fake Timers Infeasible**: Real short intervals (e.g., 100 ms ping interval) with event-driven settlement (await ping/close events), NO sleeps used for correctness. Mark feasibility of fake-timer approach as a RISK to be confirmed by the first RED spike.

**Real TLS/WSS Required**: YES, for all liveness tests (await real events on real clients; fake timers control scheduling, not ws/TLS behavior)

**No Arbitrary Sleeps**: Timing control via fake timers; event settlement via await on real events; no `setTimeout` used for correctness

**Test Home**: New describe block `'Transport Resource Controls (b1a2b)'` in tls-gateway.test.ts for liveness boundary tests; restart isolation test added to existing `'WSS gateway integration lifecycle (a2b)'` block

---

## 7. RATE LIMITING (DEFERRED)

### 7.1 Why Rate Limiting is Deferred

**"At Most One ACTIVE per station_id"**: This is peer-REPLACEMENT semantics (B1b/B1c spec requirement), NOT a rate limit. It is a stateful slot-ownership rule: when a station_id negotiates a second session, the first is replaced (sent `PEER_REPLACED`). This is implemented as part of negotiation state tracking (B1b/B1c), not a transport-level rate-control policy.

**No Authoritative Rate Threshold**: The spec provides NO numeric bound on connection attempts per station_id per time window, TLS handshake attempts per station_id per time window, or upgrade attempts per station_id per time window. B1a2b implements connection/payload/liveness CAPS (bounds on concurrency or size), not RATE windows.

### 7.2 Why IP-Based Rate Limiting is Deferred

**Real Reasons IP-Based Control Cannot Be Specified in B1a2b**:
1. **No Authoritative Requirement**: Neither the spec nor the roadmap mandates pre-identity rate limiting based on IP/MAC/hostname/mDNS.
2. **No Authoritative Threshold/Window**: No product decision exists on "X connections per IP per Y seconds" or similar.
3. **No Abuse Model Decision**: No product owner decision on whether the LAN threat model warrants IP-based rate control (untrusted key, abuse-control only, never grants authorization).

**Why NOT "Unbounded Memory is Intrinsic"**: A bounded structure with eviction (e.g., LRU map with max 1000 IPs, time-window cleanup) is technically feasible. The deferral reason is NOT that IP-based control is impossible, but that it requires explicit product/security decisions that do not exist in B1a2b's scope.

### 7.3 Smallest Defensible Surface for a LATER Slice

**Potential Follow-Up Slice** (independent of B1c, unscheduled):
- **Control**: Pre-upgrade connection/handshake attempt RATE keyed by remote address (IP:port or IP only)
- **Key Trust**: IP is an UNTRUSTED abuse-control key, NEVER used for authorization
- **Bounded Table**: Fixed-size map (e.g., 1000 entries) with LRU eviction
- **Threshold**: "X connection attempts per IP per Y seconds" (requires owner decision; illustrative examples: 10 attempts per 60 seconds—NOT proposed, only illustrative)
- **Fail-Closed Policy**: On threshold exceed, drop TCP connection before TLS handshake (or during handshake)
- **Open Decisions**: Window duration, threshold count, eviction policy (LRU, time-based, or hybrid), and whether the abuse model warrants it (LAN deployment; all clients theoretically trusted via mTLS)

**Status**: FULLY DEFERRED to a defined follow-up item (NOT "to B1c"; B1c is negotiation layer, not transport rate-control). NOT blocked on B1c. Blocked only on owner decision: what rate/window, what key, whether the LAN abuse model warrants pre-identity rate control.

### 7.4 Residual Risk Statement

**What Remains Unmitigated in B1a2b**:
1. **Rate of Connection Attempts/TLS Handshakes (CPU Churn)**: A concurrent-connection cap bounds the number of open sockets, NOT the rate at which an attacker can attempt new connections (e.g., open → close → open in a loop). Each TLS handshake consumes CPU (asymmetric crypto); a high-rate attack could degrade server responsiveness even if total concurrency is capped.
2. **Authenticated-Slot Hoarding**: An attacker with valid mTLS credentials could open `maxConnections` sockets, answer pings (stay alive), and never negotiate (hoard connection slots until B1c's handshake timeout is implemented). Heartbeat detects DEAD peers only; it is mitigated only by B1c's handshake timeout (currently an OPEN dependency).

**What IS Mitigated in B1a2b**:
- Unbounded concurrency (connection cap)
- Unbounded message size (payload cap)
- Unbounded stale-client lifetime (heartbeat liveness)

**Maintainer Acceptance Required**: Deferring rate limiting (deviation from roadmap listing it as a B1a2b control) means accepting the residual CPU churn and slot-hoarding risks until a follow-up slice. Approving this design = acceptance of those deviations and residual risks.

---

## 8. BACKPRESSURE

### 8.1 Inbound Backpressure (Implemented)

**Resource**: Inbound WebSocket frame parsing and reassembly buffers

**Protection**:
- `maxPayload`: Bounds single message size (C-category config field, B1a2b, Section 5)
- `maxBufferedChunks` (ws 8.22.0 default 262144, probed): Prevents unbounded buffering during parse (B-category ws default, version-dependent risk accepted)
- `maxFragments` (ws 8.22.0 default 16384, probed): Prevents fragmentation DoS (B-category ws default, version-dependent risk accepted)

**Status**: Implemented via ws native controls with explicit `perMessageDeflate: false` (security invariant)

### 8.2 Outbound Backpressure (B1c Concern)

**Context**: The gateway in B1a2b does NOT send application messages. The only outbound data is WebSocket pings (liveness mechanism, small 2-byte frames) and WebSocket close frames (on reject/error, one-time, small). Neither represents application-level queueing or flow control risk.

**Future Concern**: When B1c implements secure-link messages (hello, accept, reject, product actions), B1c MUST monitor `ws.bufferedAmount` before sending large messages, MAY implement message queuing if needed, and MAY apply backpressure to application layer (e.g., defer product action dispatch if outbound buffer full).

**Decision**: Outbound backpressure is deferred to B1c. B1a2b does NOT introduce outbound queues or `bufferedAmount` monitoring (no messages to queue).

---

## 9. AUDIT BOUNDARY

### 9.1 Current SecurityAudit Vocabulary

**TRANSPORT_REJECTION_REASONS** (security-audit.ts, verified by grep):
```typescript
export const TRANSPORT_REJECTION_REASONS = ['TLS_SESSION_RESUMED'] as const
```
This is a CLOSED vocabulary for transport-layer rejections (pre-negotiation, no peer identity). Policy A (TLS resumption rejection) is the only current member.

### 9.2 SecurityAuditSink Implementation Status

**Grep Result**: SecurityAuditSink interface defined in `apps/desktop/electron/security/secure-link/security-audit.ts` (line 279). Test recording sinks exist in `tls-gateway.test.ts` (lines 454, 571, 631) and `security-audit.test.ts` (line 10, 180). NO durable production sink implementation verified in Dinamizador repo (no file outside tests implements the interface with real persistence).

**Verification Summary**: SecurityAuditSink interface exists at security-audit.ts:279. Recording sinks exist only in tests (tls-gateway.test.ts, security-audit.test.ts). NO durable implementation exists in the Dinamizador repo today. WHERE and WHEN a durable sink is built is not decided by this design. Resource-event auditing would require vocabulary expansion AND sink coalescing; both are separate dependency concerns not resolved here.

### 9.3 Resource Event Audit Decision

**Question**: Should resource control events (connection cap hit, payload exceeded, liveness timeout) be audited?

**Analysis**:
1. **No Identity**: Resource controls in B1a2b are pre-identity. Audit events would lack `installationId`, `stationId`, `centerId` (all unavailable until B1b/B1c).
2. **High Frequency**: Connection cap and payload violations could be high-frequency (abuse scenarios). The spec requires the audit sink to "implement rate control/coalescing for high-frequency events". Adding resource events WITHOUT a coalescing strategy implemented in the sink creates unbounded audit load risk.
3. **No Authoritative Requirement**: Neither the spec nor roadmap mandates auditing transport resource events. Audit requirements focus on authentication, identity binding, negotiation, replay, and privilege eligibility.
4. **Vocabulary Expansion = Contract Change**: Adding new `TRANSPORT_REJECTION_REASONS` values or a separate resource event type changes the SecurityAudit contract. If done, it should be a conscious, justified expansion with a defined sink coalescing strategy, not a side effect of B1a2b.

**Decision**: B1a2b does NOT audit resource control events. The SecurityAudit contract (security-audit.ts, contracts.ts) remains unchanged. If operational monitoring later requires resource event tracking, it can be added as a separate monitoring/metrics system (not SecurityAudit) or a future SecurityAudit expansion (explicit design decision with coalescing strategy proven in the sink).

---

## 10. CONFIGURATION CONTRACT & VALIDATION

### 10.1 TlsGatewayConfig Changes

**Existing** (baseline tls-gateway.ts):
```typescript
export interface TlsGatewayConfig {
  readonly serverCert: Buffer
  readonly serverKey: Buffer
  readonly trustedCA: Buffer
  readonly host: string
  readonly port: number
  readonly auditSink?: SecurityAuditSink
}
```

**New** (B1a2b):
```typescript
export interface TlsGatewayConfig {
  readonly serverCert: Buffer
  readonly serverKey: Buffer
  readonly trustedCA: Buffer
  readonly host: string
  readonly port: number
  readonly auditSink?: SecurityAuditSink
  // B1a2b: Resource controls (all REQUIRED, validated as finite positive safe integers)
  readonly maxConnections: number
  readonly maxPayload: number
  readonly pingIntervalMs: number
}
```

### 10.2 Validation Requirements

**Validation**: Gateway constructor MUST validate B1a2b fields as finite, positive, safe integers. Throw synchronously with a descriptive error if any field is `undefined`, `null`, `NaN`, `Infinity`, negative, zero, or not a safe integer.

**Validation Logic** (per field):
```typescript
if (!(Number.isSafeInteger(config.maxConnections) && config.maxConnections > 0)) {
  throw new Error('maxConnections must be a finite positive integer')
}
// Similar for maxPayload, pingIntervalMs
```

**Synchronous Throw on Invalid Config**: `createTlsGateway(config)` throws immediately (before any resource is created, before `pendingStart` is set) if validation fails. This is a CHANGE to the factory contract (previously did not throw on construction).

**Config Discipline**: The three C-category thresholds (`maxConnections`, `maxPayload`, `pingIntervalMs`) are REQUIRED fields with NO hidden production defaults. Test helpers provide example values; production deployment blocks on owner selection of operational thresholds.

---

## 11. TDD PROOF PLAN

### 11.1 Test Structure

**New Describe Block**: `'Transport Resource Controls (b1a2b)'` in tls-gateway.test.ts

**Test Strategy**: Real TLS/WSS for all boundary proofs (connection cap, payload, liveness dead-peer); fake timers for liveness policy tests (interval cleanup, restart isolation); unit-level config validation; event-driven settlement (no arbitrary sleeps).

### 11.2 Test Inventory

**Config Validation** (1 parameterised it.each test, ~15 lines, unit-level):
- Lives in new `'Transport Resource Controls (b1a2b)'` block
- 3 test cases via it.each: missing maxConnections, non-positive maxPayload, non-integer pingIntervalMs

**Connection Cap** (1 test, ~27 lines, integration-level real TLS/WSS):
- `rejects third connection when maxConnections=2 and releases slot`
- Lives in new `'Transport Resource Controls (b1a2b)'` block

**Payload Limit** (1 test, ~27 lines, integration-level real TLS/WSS):
- `terminates client on payload exceeding maxPayload and server stays healthy`
- Lives in new `'Transport Resource Controls (b1a2b)'` block

**Liveness Mechanism** (4 tests, ~111 lines total, integration-level real TLS/WSS + fake timers):
1. `pings responsive client without terminating over multiple intervals` (~30 lines; includes ordering barrier: after client observes server's ping, client sends client.ping() and awaits its own pong event before advancing next tick, ensuring server has processed the auto-pong before time advances; prevents pong-in-flight race that could flaky-terminate a healthy client; no sleeps used)
2. `terminates unresponsive client after two intervals` (~27 lines)
3. `interval cleanup/getTimerCount` (~27 lines)
4. `restart isolation` (~27 lines)
- All 4 tests live in new `'Transport Resource Controls (b1a2b)'` block

**Call-Site Updates** (~11 lines):
- Update test helper and inline configs across tls-gateway.test.ts (~8 lines) and tls-gateway-startup.test.ts (~3 lines) to include the three new required fields
- Counted in test forecast (section 12.2) but not as separate tests

Total new tests: +9 vitest tests, all in the new 'Transport Resource Controls (b1a2b)' block (3 config-validation cases from one parameterised it.each + 1 connection cap + 1 payload + 4 liveness); +0 in the existing lifecycle block. New tls-gateway.test.ts total: 53 + 9 = 62; tls-gateway-startup.test.ts unchanged at 1.

**New Baseline**: 53 + 9 = **62 tests**

### 11.3 Fallback Strategy for Fake Timers

**Primary Strategy**: Narrow fake timers (`toFake: ['setInterval', 'clearInterval']`) with `vi.getTimerCount()` to observe interval creation/cleanup.

**Fallback if Infeasible**: Real short intervals (e.g., 100 ms ping interval) with event-driven settlement (await ping/close events), NO sleeps for correctness. Mark feasibility as a RISK to be confirmed in the FIRST RED spike (liveness test setup). If fake timers prove infeasible, the fallback still provides deterministic behavioral proof without arbitrary timing dependencies.

### 11.4 Test-Count Delta

**Baseline**: tls-gateway.test.ts currently has 53 tests (verified by grep: 10 + 2 + 4 + 8 + 1 + 10 + 9 + 9)

**New Baseline**: 53 + 9 = **62 tests**

**Other Files**: tls-gateway-startup.test.ts unchanged (1 test, 37 lines)

---

## 12. FILE CHANGES & SIZE FORECAST

### 12.1 Production Code Changes (tls-gateway.ts)

**File**: apps/desktop/electron/security/transport/tls-gateway.ts (baseline 321 lines)

**Changes**:
1. **Config Validation** (~12 lines): Validate `maxConnections`, `maxPayload`, `pingIntervalMs` in `createTlsGateway` before any resource creation
2. **Connection Cap** (~1 line): Set `nextServer.maxConnections = config.maxConnections`
3. **Payload Limits** (~3 lines): Add `maxPayload` and `perMessageDeflate: false` to WebSocketServer constructor options
4. **Liveness Mechanism** (~46 lines):
   - `start()` creates a LOCAL generation object (its own awaitingPong Set and interval handle) that the interval callback closes over (plus shared activeClients/lifecycle)
   - `startPingLoop()` function (~20 lines): create generation, create interval, tick body with try/catch, lifecycle guard, iterate generation.awaitingPong (terminate), clear, iterate activeClients (add + ping)
   - `stopPingLoop()` function (~3 lines): clearInterval on current generation's handle, clear generation Set
   - Gateway keeps a reference to the current generation needed by stop(); stale callbacks cannot act on new generation's clients
   - Pong handler attachment (~2 lines): `client.on('pong', () => awaitingPong.delete(client))`
   - Integration (~6 lines): call `startPingLoop()` after lifecycle = RUNNING, call `stopPingLoop()` only from `stop()` (a failed start never creates an interval, verified by getTimerCount()==0 in the failed-start test)
   - Initialization (~2 lines): initialize closure-scoped liveness state

**Estimated Net Production Lines**: ~62 lines (validation 12 + connection cap 1 + payload/compression 3 + liveness 46)

**Production Forecast Range**: 62–78 lines (point estimate: 70 lines)

### 12.2 Test Changes

**File**: apps/desktop/test/main/security/transport/tls-gateway.test.ts (baseline 1026 lines)

**Call-Site Updates** (~8 lines):
- Update `config()` helper (line ~813) to add three new required fields: `maxConnections: 10, maxPayload: 1048576, pingIntervalMs: 5000` (~3 lines change in the helper definition)
- Update Policy A test inline config (line ~34-40) to add the three fields (~3 lines)
- Update other inline configs if any (margin: ~2 lines)

**New Tests** (~180 lines):
- Config validation: 1 parameterised it.each test, ~15 lines
- Connection cap: 1 real-WSS test, ~27 lines
- Payload limit: 1 real-WSS test, ~27 lines
- Liveness mechanism: 4 real-WSS tests:
  - Responsive client (with ordering barrier): ~30 lines
  - Unresponsive client: ~27 lines
  - Interval cleanup/getTimerCount: ~27 lines
  - Restart isolation: ~27 lines
- Real-WSS test density verified from PR-08B1a2a2 a2b: 7 tests added 186 net lines = 26-27 lines/test

**Estimated Test Net Lines**: ~191 lines (new tests 180 + call-site updates 11)

**Test Forecast Range**: 180–210 lines (point estimate: 195 lines; based on observed 26-27 lines/test density for real-TLS/WSS tests)

### 12.3 Other Files

**File**: apps/desktop/test/main/security/transport/tls-gateway-startup.test.ts (baseline 37 lines)

**Call-Site Update** (~3 lines): Add three required fields to inline config (line ~24-28)

**Estimated Net Lines**: ~3 lines

### 12.4 Total Size Forecast

**Production**: 62–78 lines (point: 70)
**Tests**: 180–210 lines (point: 195)
**Test call-site updates**: ~11 lines (counted in test forecast above; broken out for clarity: ~8 in tls-gateway.test.ts, ~3 in tls-gateway-startup.test.ts)
**Total Point Estimate**: 70 + 195 = **265 net lines** (production + tests including call-site updates)

**Risk-Adjusted Upper Bound**:
- Historical overrun: PR-08B1a2a2 (a2b) forecast ~262 net, landed 367 (+40%)
- Apply 40% overrun to point estimate: 265 × 1.4 = **371 net lines**
- Risk-adjusted upper bound: **371 net lines**
- Margin to 400-line gate: **29 lines** (thin, tens of lines)

**Review Budget Gate**: 400 net changed lines (canonical threshold)

**Single-Slice Decision with Checkpoint**: The risk-adjusted upper bound (371 net lines) is within the 400-line budget but with thin margin (29 lines). **SINGLE SLICE APPROVED with MANDATORY CHECKPOINT**: After the static-limits phase (config contract + connection cap + payload + perMessageDeflate:false) passes GREEN, measure cumulative net lines added. If the projection to complete all in-scope controls (connection cap, payload, heartbeat) exceeds 380 net lines, the implementer MUST STOP and report to the parent/maintainer (no automatic split or exception). If the checkpoint is exceeded, the fallback split strategy (Section 12.5) is a PROPOSAL for that conversation, NOT an automatic action.

### 12.5 Fallback Split Strategy (IF Gate Exceeded)

**IF** the implementation exceeds the 400 NET line gate during RED-first development, the implementer MUST STOP and report to the parent/maintainer. NO automatic split or exception. The pre-defined fallback seam (for that conversation only):

**Slice 1 (Static Limits, B1a2b1)**: Connection cap + payload limit + config validation + explicit perMessageDeflate:false (~101-120 net lines: 16-20 production + 85-100 tests including call-site updates)

**Slice 2 (Liveness, B1a2b2)**: Heartbeat ping/pong mechanism (pingIntervalMs + awaiting-pong + timer lifecycle) (~163-197 net lines: 46-60 production + 117-137 tests including call-site updates)

**Corrected Forecasts Based on 26-27 Lines/Test Density**: Static limits ~85-100 test lines (config 15 + connection cap 27 + payload 27 + call-sites 4); liveness ~117-137 test lines (4 real-WSS tests at 27-30 lines each + call-sites 7).

This split is a PROPOSAL only, to be negotiated with the maintainer if the gate is exceeded. It is NOT an automatic fallback.

---

## 13. IMPLEMENTATION ORDER (RED-First TDD)

### 13.1 RED-First Discipline

Each control follows strict TDD: write the failing test FIRST (RED), verify it fails for the right reason, then implement the minimal production code to make it pass (GREEN), then refactor if needed (REFACTOR).

### 13.2 Implementation Sequence

**Phase 1: Config Validation** (RED-first):
1. RED: Write test `throws on missing maxConnections` → run → verify fails (no validation exists)
2. GREEN: Add validation logic for `maxConnections` → run → pass
3. RED: Write test `throws on non-positive maxPayload` → run → verify fails
4. GREEN: Add validation logic for `maxPayload` → run → pass
5. RED: Write test `throws on non-integer pingIntervalMs` → run → verify fails
6. GREEN: Add validation logic for `pingIntervalMs` → run → pass
7. REFACTOR: Extract validation helper if needed
8. Update all existing test call sites to provide valid config fields (tests fail until this is done)

**Phase 2: Connection Cap** (RED-first):
1. RED: Write test `rejects third connection when maxConnections=2 and releases slot` → run → verify 3rd client opens (no limit enforced)
2. GREEN: Set `nextServer.maxConnections = config.maxConnections` in `start()` → run → pass

**Phase 3: Payload Limit** (RED-first):
1. RED: Write test `terminates client on payload exceeding maxPayload and server stays healthy` → run → verify large message accepted (no limit enforced)
2. GREEN: Add `maxPayload: config.maxPayload` and `perMessageDeflate: false` to WebSocketServer options → run → pass

**Phase 4: Liveness Mechanism** (RED-first, most complex):
1. RED: Write test `terminates unresponsive client after two intervals` (with fake timers, `autoPong: false` client) → run → verify client NOT terminated (no ping loop exists)
2. GREEN: Implement minimal ping loop (closure-scoped liveness state, startPingLoop/stopPingLoop, interval creation/clearing, awaiting-pong pattern, terminate unresponsive clients, integrate into start/stop) → run → pass
3. Regression/characterisation guard: Write test `pings responsive client without terminating over multiple intervals` → run → expected GREEN (proves liveness does not disturb responsive clients; regression guard, not RED evidence)
4. REFACTOR: Extract ping loop logic, improve error handling, ensure lifecycle guards are correct

**Phase 5: Lifecycle/Restart Cleanup** (RED-first):
1. RED: Write test `liveness mechanism survives restart without timer leaks` (fake timers, `vi.getTimerCount()` assertions, restart isolation) → run → verify fails (timer leak or old interval affects new generation)
2. GREEN: Ensure `stopPingLoop()` clears interval correctly, `startPingLoop()` creates fresh interval, closure isolation prevents stale callbacks → run → pass
3. REFACTOR: Verify no global/module-level state, all liveness state is closure-scoped

**Phase 6: Regression & Integration**:
1. Run ALL existing tests (53 baseline + 9 new = 62 total) → verify no regressions
2. Run full test suite for tls-gateway.test.ts and tls-gateway-startup.test.ts → verify all pass
**Fallback Risk Confirmation** (during Phase 4, step 1):
- If fake timers prove infeasible (Vitest limitations with narrow faking + `vi.getTimerCount()`), pivot to fallback strategy: real short intervals (100 ms) with event-driven settlement (await ping/close events), NO sleeps for correctness. Report the pivot to the parent/maintainer.

---

## 14. DECISIONS RESOLVED / OPEN / BLOCKERS

### 14.1 Architecture Decisions Resolved by Approving This Design

Approving this design = maintainer acceptance of:

1. **Config Discipline**: All C-category thresholds (`maxConnections`, `maxPayload`, `pingIntervalMs`) are REQUIRED config fields with NO hidden production defaults. Production wiring is blocked on owner numeric values (see Section 14.3).
2. **Rate Limiting Deferral**: Rate limiting is fully deferred to an unscheduled independent follow-up slice (NOT blocked on B1c). This deviates from the roadmap wording that lists rate limiting as part of the same resource-hardening initiative. Residual risk: CPU churn from high-rate connection attempts and authenticated-slot hoarding (see Section 7.4).
3. **Idle Timeout Absorption**: "Idle timeout" (as listed separately in the roadmap) is absorbed into heartbeat dead-peer detection at the transport level. Application-level idle timeout (upgraded-but-never-negotiates) is deferred to B1c's handshake timeout. This deviates from the roadmap wording. Residual risk: authenticated peers can hoard slots until B1c delivers its handshake timeout (see Section 6.1).
4. **Single-Threshold Liveness**: The awaiting-pong pattern uses ONE operational threshold (`pingIntervalMs`); no second `pongTimeoutMs` threshold. Dead-peer detection latency is 1-2x `pingIntervalMs` (deterministic, simple, no per-client timestamps).
5. **Explicit perMessageDeflate:false**: Compression is explicitly disabled (not relying on ws library default) to prevent compression side-channel attacks (CRIME/BREACH-class) and ensure immutability across ws upgrades.
6. **Single Slice with Checkpoint**: The risk-adjusted upper bound (371 net lines) is within the 400-line review budget but with thin margin (29 lines). SINGLE SLICE APPROVED with MANDATORY CHECKPOINT: after the static-limits phase (config contract + connection cap + payload + perMessageDeflate:false) passes GREEN, measure cumulative net lines; if the projection to complete all in-scope controls (connection cap, payload, heartbeat) exceeds 380 net lines, the implementer MUST STOP and report (no automatic split or exception).
7. **No Audit Expansion**: Resource control events (connection cap hit, payload exceeded, liveness timeout) are NOT audited. SecurityAudit contract unchanged.

### 14.2 Owner Values Still Open (Block Production Wiring, NOT Implementation)

The following numeric values are CONFIG REQUIRED and block PRODUCTION WIRING (B1c composition root), NOT implementation or merge of B1a2b:

1. **maxConnections** (integer): Maximum concurrent raw TCP sockets (pre-TLS, pre-upgrade, accepted WSS). Operational security/capacity trade-off. Must account for expected half-open socket lifetime (up to 2 minutes for `handshakeTimeout`) and reconnect overlap (up to 2x `pingIntervalMs`).
2. **maxPayload** (integer, bytes): Maximum WebSocket message payload size. Operational DoS threshold; no spec-defined bound.
3. **pingIntervalMs** (integer, milliseconds): Liveness probe interval. Dead-peer detection latency is 1-2x this value. Network/responsiveness trade-off.

**Test Values** (for reference, NOT proposed for production):
- Tests use `maxConnections: 10` (generous enough for existing tests; connection cap boundary test uses its own small value 2)
- Tests use `maxPayload: 1048576` (1 MiB; payload boundary test uses its own small value 16)
- Tests use `pingIntervalMs: 5000` (5 seconds; liveness tests may use shorter intervals for speed)

**Production Deployment**: Blocked on product owner decision for the three C-category thresholds. This is a PRODUCTION WIRING blocker, NOT an implementation gate (tests provide their own values).

### 14.3 Production-Wiring Blockers (B1c Composition Root)

**B1c Integration Blockers**:
1. **Owner Numeric Values** (see Section 14.2): `maxConnections`, `maxPayload`, `pingIntervalMs` must be decided before B1c can construct the gateway in the production composition root.
2. **B1c Handshake Timeout** (dependency): B1c MUST implement an application-level handshake timeout (e.g., "terminate if no hello within X seconds of upgrade") to close the authenticated-slot-hoarding gap (peers that upgrade, answer pings, but never negotiate). B1c owns that timeout and its value; B1a2b emits NO wire code for liveness/idle termination.

**B1a2b Does NOT Block B1c Development**: B1c can proceed with message handling, FSM, negotiation in parallel. The integration point is the gateway construction in the composition root (when B1c is ready to start the gateway with production config).

### 14.4 Cross-Slice Dependencies

**B1a2b → B1c**:
- B1c MUST implement handshake timeout (see Section 14.3)
- B1c MUST monitor `ws.bufferedAmount` if sending large messages (outbound backpressure; see Section 8.2)

**B1a2b → Usuario PC**:
- Usuario PC WebSocket client MUST maintain an active read loop to answer server pings (cross-repo conformance dependency; see Section 3)

**B1a2b → Future Rate Limiting Slice** (if pursued):
- Owner decision on rate threshold/window, key trust assumptions, whether LAN abuse model warrants it (see Section 7.3)

**NO Dependency on B1b**: B1a2b does NOT require X.509 identity extraction (B1b). Connection limits, payload limits, and liveness are pre-identity transport controls.

---

## 15. CONCISE RISKS

### 15.1 Residual Security/Operational Risks

1. **Half-Open Socket Starvation**: Native timeouts (`handshakeTimeout` up to 2 minutes) bound half-open socket lifetime, but operator must size `maxConnections` with margin for expected half-open sockets and reconnect overlap (see Section 4.2).
2. **Authenticated-Slot Hoarding**: Peers with valid mTLS credentials can hoard slots by answering pings but never negotiating (no hello sent). Heartbeat detects DEAD peers only, not idle peers. Mitigated only by B1c's handshake timeout (OPEN dependency; see Section 6.1).
3. **CPU Churn from High-Rate Connection Attempts**: A concurrent-connection cap bounds concurrency, NOT the rate of connection attempts. Repeated TLS handshakes (open → close → open loop) can degrade server responsiveness. Unmitigated in B1a2b; deferred to future rate-limiting slice (see Section 7.4).
4. **Version-Dependent Defaults**: Reliance on ws defaults (`maxBufferedChunks`, `maxFragments`) and Node.js defaults (handshake/header/request timeouts) is implicit and version-dependent. Accepted risk with revisit trigger on ws/Node.js upgrades (see Section 2).

### 15.2 Implementation Risks

1. **Fake Timer Feasibility**: Narrow fake timers (`toFake: ['setInterval', 'clearInterval']`) with `vi.getTimerCount()` may prove infeasible in Vitest. Fallback strategy exists (real short intervals, event-driven settlement), but feasibility must be confirmed in the FIRST RED spike (liveness test setup; see Section 11.3).
2. **Test Overrun**: Historical overrun (+40% on PR-08B1a2a2 a2b) applied to risk-adjusted upper bound (371 net lines). Margin to 400-line gate is thin (29 lines). MANDATORY CHECKPOINT after static-limits phase GREEN: if projection exceeds 380 net lines, implementer MUST STOP and report to maintainer (no automatic exception; see Section 12.4).

### 15.3 Cross-Repo Risks

1. **Usuario PC Client Conformance**: Assumes Usuario PC WebSocket client maintains an active read loop to answer server pings. If the client disables auto-pong or does not read from the socket, heartbeat liveness will falsely detect it as a dead peer. Verification deferred to cross-repo integration testing (see Section 3).

---

## 16. IMPLEMENTATION GATE

### 16.1 Gate Conditions (Must Be TRUE to Proceed)

This design is ready for implementation if ALL of the following are TRUE:

1. **Architecture Approval**: Maintainer approves the architecture decisions in Section 14.1 (config discipline, rate limiting deferral, idle timeout absorption, single-threshold liveness, explicit perMessageDeflate:false, single slice, no audit expansion).
2. **Residual Risk Acceptance**: Maintainer accepts the residual risks in Section 15.1 (half-open starvation, authenticated-slot hoarding, CPU churn, version-dependent defaults) as acceptable until future slices (B1c handshake timeout, potential rate-limiting slice).
3. **Test Strategy Approval**: Maintainer approves the TDD proof plan (Section 11) and fallback strategy for fake timers (Section 11.3).
4. **NO Owner Numeric Values Required**: Implementation and merge of B1a2b do NOT require owner numeric values for `maxConnections`, `maxPayload`, `pingIntervalMs`. Tests provide their own values. Production wiring (B1c composition root) IS blocked on owner values, but that does NOT block B1a2b implementation/merge.

### 16.2 Gate Does NOT Require

The implementation gate does NOT require:

1. **Owner Numeric Values**: `maxConnections`, `maxPayload`, `pingIntervalMs` are CONFIG REQUIRED and block production wiring, NOT implementation/merge (see Section 14.2).
2. **B1c Handshake Timeout**: This is a B1c dependency for closing the authenticated-slot-hoarding gap, NOT a B1a2b implementation blocker (see Section 14.3).
3. **Rate Limiting Design**: Rate limiting is fully deferred (see Section 7); its absence does NOT block B1a2b.
4. **Audit Sink Implementation**: NO durable SecurityAuditSink exists in Dinamizador repo; B1a2b does NOT audit resource events (see Section 9).

### 16.3 Stop Condition During Implementation

**MANDATORY CHECKPOINT**: After the static-limits phase (config contract + connection cap + payload + perMessageDeflate:false) passes GREEN, measure cumulative net lines added. If the projection to complete all in-scope controls (connection cap, payload, heartbeat) exceeds 380 net lines, the implementer MUST STOP and report to the parent/maintainer. NO automatic split or exception. The fallback split strategy (Section 12.5) is a PROPOSAL for that conversation, NOT an automatic action.

**IF** the implementation exceeds the 400 NET line review budget at any point, the implementer MUST STOP and report to the parent/maintainer. The risk-adjusted upper bound (371 net lines) provides thin margin (29 lines) to the 400-line gate.

---

## ARCHITECT VERDICT: APPROVE
