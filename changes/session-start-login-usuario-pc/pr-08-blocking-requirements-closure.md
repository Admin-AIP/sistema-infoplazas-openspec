# PR-08 — Cierre aprobado de requisitos bloqueantes

> **Resultado autoritativo:** los nueve bloqueadores técnicos originales están
> **CLOSED** por decisiones de arquitectura aprobadas. El contrato queda listo
> para planificación de implementación, pero este cierre no autoriza `apply`,
> worktrees, cambios de producto ni acciones Git.

## Ruta de revisión

1. Verificar la matriz de cierre de los nueve bloqueadores.
2. Revisar el contrato normativo cruzado en
   [`specs/dinamizador-usuario-pc-secure-link/spec.md`](./specs/dinamizador-usuario-pc-secure-link/spec.md).
3. Confirmar el límite de cero acciones de producto y la subdivisión obligatoria.
4. Ejecutar un preflight nuevo antes de la siguiente slice técnica: PR-08B1a2b.

## 1. Jerarquía y naturaleza de las decisiones

Las fuentes se conservan en este orden: PDF del Dinamizador; PDF de Usuario PC;
DOCX general; especificaciones derivadas; decisiones arquitectónicas y
`guardrails` aprobados; código; historia técnica.

Los PDFs exigen estaciones registradas/autorizadas, vínculo con su centro,
operación LAN sin Internet permanente, consideración del ciclo de
vida/recuperación y auditoría. No se atribuye a los PDFs una selección
criptográfica concreta ni un algoritmo anti-replay. WSS sobre TLS 1.3 con mTLS,
la PKI privada, la negociación v2 y las reglas de época/secuencia son decisiones
de **ARQUITECTURA TÉCNICA aprobadas**.

### Estado integrado autoritativo de PR-08

Dinamizador `master == origin/master` está en
`eef27518862506a52f1f40dab5c870701d1be5d4`. PR-08A1 está CLOSED como A1a
`83fd2fc4a57f469c99fc36464a9a0af7084f70c0` + A1b
`9216370ef4876d553e0acef18d16821ddcd11cee`. PR-08A2 está CLOSED como A2a
`1dbd4eafaacf9b10f91b944b32a9451e699f43bf`, A2b
`efa8c84eecdc116d6ff4a455fd8d82ecdd529d20`, A2c1
`9d81269e2f6ee49590c182298c157c50449b0f32` y A2c2
`6a51a1409ee52368d938b0eaead5bfcce479d63c`.

PR-08B1 está **IN PROGRESS**. B1a1 está CLOSED en
`de6669ad2e511ec8fd51e5db6a29fdcf1583e944`. B1a2a1f, solo fixtures TEST-ONLY
sin secretos de producción ni clave privada firmante de CA, está
CLOSED/INTEGRATED/PUSHED en `22f509c88412bba53d39361dc7aa78aee1806f47`.
B1a2a1 está CLOSED/INTEGRATED/PUSHED en
`2b713ce450a295a996c8dad2b4b1d0957c6db99a`, con parent
`22f509c88412bba53d39361dc7aa78aee1806f47`: 299 líneas reales, exactamente 102
de producción + 197 de pruebas. Entrega base HTTPS, TLS 1.3-only, rechazo TLS
1.2, mTLS con `requestCert: true` y `rejectUnauthorized: true`, confianza
exclusiva en CA privada del proyecto, pruebas de
cliente confiable/ausente/no confiable, lifecycle y manejo de error runtime. Su
evidencia es TLS 6/6, secure-link 90/90, security/main 101/101 y typecheck PASS.
`SSL_OP_NO_TICKET` está configurado, pero **no prueba** ausencia de reanudación
(evidencia histórica de B1a2a1; resuelta después por Policy A, ver más abajo).

**Policy A — resolución de reanudación TLS — está CLOSED / INTEGRATED**
(B1a2a1r). Secuencia integrada: núcleo TLS B1a2a1
`2b713ce450a295a996c8dad2b4b1d0957c6db99a`; rechazo de reanudación r1
`b680e5c3d94ca57b566d1b8cbd7512aa54e706ae`; modelo de auditoría de transporte
r2a `77143cccf61d92066b0742d5dcfef06404a55924`; endurecimiento de la frontera
de auditoría r2a1 `0b51f8fdef45592ccd85750ace3bb36f386f98a4`; cableado de
auditoría de reanudación r2b `4cc81696b0b8f9de4868029f54b7abbb9197b9ae`.
Propiedad de seguridad: ADA no acepta una conexión de transporte TLS reanudada.
Policy A no se reabre.

**PR-08B1a2a2 está CLOSED / INTEGRATED**, dividido semánticamente. Capacidad
final: transporte WSS seguro + lifecycle seguro del gateway.

- **PR-08B1a2a2a — WSS Validation Primitives & Dependencies — CLOSED /
  INTEGRATED**, commit `1e1a11ea5b6d2ff5209ca673a639dd1877605202`. Entrega la
  dependencia `ws`, `@types/ws`, `containsToken()`, `validateUpgradeRequest()`
  y `validatePeerCertificateDates()`. Revisión: native review APPROVED;
  Gemini APPROVE.
- **PR-08B1a2a2b — WSS Integration & Safe Gateway Lifecycle — CLOSED /
  INTEGRATED**, commits `780ab0435c87aada50d537bbd4847dca560f2de1` y
  `eef27518862506a52f1f40dab5c870701d1be5d4` (HEAD integrado). Entrega
  `WebSocketServer({ noServer: true })`, integración de HTTPS Upgrade, WSS con
  TLS 1.3 + mTLS, validación temporal y validación de HTTP Upgrade antes de
  `handleUpgrade`, propiedad de WebSockets activos, rollback seguro de arranque
  fallido, coordinación de stop durante start, terminación determinista de
  clientes, aislamiento de restart y contención de frames malformados.
  Revisión: Gemini APPROVE; revisión nativa en estado final
  **ESCALATED — MAINTAINER OVERRIDE** (no aprobada, no superada y no limpia;
  el detalle está en [`design.md` de PR-08B1a2a2](../PR-08B1a2a2/design.md)).

**PR-08B1a2b — NEXT / PREFLIGHT REQUIRED / NOT STARTED.** Es control de
recursos/hardening de transporte (límites de conexión, límites de payload, idle
timeout, heartbeat, rate limiting y backpressure). Requiere un preflight fresco
de arquitectura, alcance y tamaño ≤400 antes de cualquier implementación; no
está lista para implementar. B1b está PLANNED / BLOCKED por la codificación de
identidad de negocio X.509; B1c está PLANNED / NOT STARTED.

El `master` actual contiene `ws`, WSS con TLS 1.3 + mTLS, integración de HTTP
Upgrade y la resolución de reanudación TLS (Policy A). El `master` actual NO
contiene —y permanecen ausentes o diferidos— controles de recursos B1a2b,
identidad de negocio, autorización de instalación/centro/deny, composición
completa transporte-FSM, ACTIVE de producción ni acciones de producto. Las
acciones de producto son **CERO** (`Product actions: ZERO`).

## 2. Matriz de cierre de los nueve bloqueadores

| # | Bloqueador original | Estado | Decisión aprobada que lo cierra |
| ---: | --- | --- | --- |
| 1 | Mecanismo de credencial y autenticación | **CLOSED** | Tráfico LAN privilegiado confiable usa WSS sobre TLS 1.3 con mTLS. Usuario PC es cliente WSS y Dinamizador es servidor WSS. Se prohíben WS en claro, contraseña global/compartida y handshake criptográfico propio. |
| 2 | Autoridad y ceremonia de enrolamiento | **CLOSED** | CA privada controlada por Infoplazas; claves generadas en dispositivo; CSR con emisión online autorizada o paquete offline firmado; trust root público fijado localmente; identidad/certificado por instalación y registro autorizado vinculado a centro. |
| 3 | Principal del Dinamizador | **CLOSED** | Cada instalación de Dinamizador posee identidad de máquina y certificado propios, registro autorizado y binding a un centro. La identidad del operador es distinta y posterior. |
| 4 | Ciclo de vida, revocación, reinstalación y reemplazo | **CLOSED** | Certificados finitos; renovación/reemisión distinta de rotación de clave/nuevo enrolamiento; reinstalación/reemplazo genera clave nueva y deshabilita/revoca la anterior; overlap solo para cutover explícito de una instalación; deny local fail-closed y sincronización central cuando exista. |
| 5 | Prueba y schema del handshake | **CLOSED** | TLS 1.3 aporta prueba mutua y protección de transcript; 0-RTT deshabilitado. Capa 2 reutiliza PR-07 con `link.hello`, `link.accept` y `link.reject`, schemas mínimos exactos y sin desafío criptográfico propio. |
| 6 | Registro de capacidades/versiones | **CLOSED** | Protocolo exacto `2`, payload schema `1`, registro explícito allowlisted y capacidad obligatoria `secure_link_v1`. PR-08 no anuncia capacidades de negocio; desconocidas otorgan cero privilegio y no existe downgrade de seguridad a v1. |
| 7 | Época, secuencia y frescura | **CLOSED** | Dinamizador genera `connection_epoch` criptográficamente aleatorio, no cero, `uint64`, nuevo por conexión y no persistido. Secuencia direccional `uint64` comienza en 1, exige `+1`, cierra ante gap/overflow y se reinicia solo bajo época nueva. `sent_at` es observacional; validez X.509 usa reloj OS local. |
| 8 | Taxonomía de rechazo | **CLOSED** | Se separan categorías de auditoría TLS local —sin JSON— de categorías wire posteriores a mTLS. Un TLS alert no es `link.reject`; `REJECTED` no implica que se haya enviado frame. La allowlist aprobada no porta secretos. |
| 9 | Durabilidad y conducta de auditoría | **CLOSED** | `SecurityAuditSink` local durable con allowlist mínima. Fallo del sink nunca concede privilegio; si auditoría obligatoria no está disponible, elegibilidad privilegiada permanece falsa, aunque el vínculo pueda seguir técnicamente autenticado para operación no privilegiada. Retención/acceso/exportación siguen como política de negocio separada. |

La matriz cierra únicamente estos nueve requisitos técnicos. No cierra vigencia
offline de credenciales de usuario, intentos/bloqueo de PIN, umbrales de
interrupción, políticas de modalidad, catálogos, reglas de sesión, conciliación,
privacidad, contenido de widget ni captura de PIN en
`DIRECT_DINAMIZADOR_RESET`.

## 3. Transporte, PKI y almacenamiento protegido

- El único transporte confiable para tráfico LAN privilegiado es WSS sobre TLS
  1.3 con autenticación mutua.
- La clave privada de firma de la CA nunca reside en estaciones ni en
  instalaciones Dinamizador.
- Cada instalación tiene certificado e identidad propios; nunca se comparten
  entre estaciones, centros o instalaciones Dinamizador.
- La autenticación runtime funciona completamente offline con credenciales no
  expiradas, raíz pública fijada localmente y estado local conocido.
- Las claves privadas se guardan en Windows con ACL restrictiva. Un spike elige
  Certificate Store/CNG cuando sea práctico o almacenamiento de aplicación
  protegido con DPAPI.
- El spike MUST probar que la clave privada nunca sale de la instalación. El
  adaptador DPAPI solo es aceptable si no presenta ruta de texto plano,
  exportación o copia y ofrece protección equivalente; de lo contrario falla.
  Este cierre no selecciona adaptador.

## 4. Identidad y autorización de instalación

Una identidad confiable exige conjuntamente:

1. certificado válido y localmente confiable;
2. registro de instalación autorizado y no deshabilitado;
3. binding exacto al `center_id` autorizado.

Esto aplica a estación y Dinamizador. `station_id`, `center_id`, `source_id` y
`link_id` del sobre siguen siendo claims hasta compararse con certificado,
registro y binding de centro. IP, MAC, hostname, nombre visible y descubrimiento
mDNS no confieren identidad. La identidad humana del operador permanece
separada y se evaluará después para autorización de workflow.

## 5. Enrolamiento y ciclo de vida offline

- La clave se genera en el dispositivo y el CSR se aprueba mediante emisión
  online autorizada o paquete de provisionamiento offline firmado.
- La raíz/CA pública se fija localmente; la clave firmante permanece fuera de
  endpoints.
- Los certificados tienen vigencia finita. Renovación/reemisión de certificado
  no equivale a rotar la clave privada; rotación genera clave y enrolamiento
  nuevos.
- Reinstalación o reemplazo siempre genera credencial nueva y deshabilita/revoca
  la anterior.
- Un overlap temporal, si operación lo exige, representa una sola instalación
  bajo cutover explícito. La credencial anterior se niega al completar cutover o
  al expirar.
- Ambos endpoints retienen peers autorizados/deshabilitados. Dinamizador mantiene
  específicamente un registro local deny/de estaciones deshabilitadas.
- Revocación central sincroniza cuando hay conectividad. Revocación global
  inmediata es imposible durante aislamiento totalmente offline; la limitación
  se acepta y se documenta. Estado local conocido como inválido, revocado o
  deshabilitado falla cerrado.
- PR-08 no fija cadencia de rotación.

## 6. FSM y autenticación TLS

```text
DISCONNECTED
  → CONNECTED_UNAUTHENTICATED   (solo TCP)
  → AUTHENTICATING              (TLS 1.3 mTLS + upgrade WebSocket)
  → AUTHENTICATED_NEGOTIATING   (WSS/mTLS válido; registro/centro/capacidades pendientes)
  → ACTIVE

cualquier estado aplicable → REJECTED o CLOSING → DISCONNECTED
```

Cada conexión o reconexión empieza sin autenticación, época, secuencia,
capacidades ni privilegios heredados. Reemplazar el peer invalida inmediatamente
el binding. TLS 1.3 early data/0-RTT queda deshabilitado. La reanudación TLS solo
puede usarse si en cada conexión nueva se revalidan vigencia del certificado,
identidad autenticada, registro autorizado, centro y deny local; si el stack no
lo garantiza, se deshabilita la reanudación. La política quedó resuelta como
**Policy A — CLOSED / INTEGRATED** (B1a2a1r): B1a2a1 configuró
`SSL_OP_NO_TICKET`, pero observó material de evento/ticket TLS 1.3 bajo Node
22/OpenSSL 3 sin probar ausencia de reanudación, por lo que Policy A rechaza
estructuralmente toda conexión reanudada antes de cualquier procesamiento
HTTP/Upgrade/WSS y audita el rechazo como evento de transporte.
`SSL_OP_NO_TICKET` se conserva como defensa en profundidad. ADA no acepta una
conexión de transporte TLS reanudada.

## 7. Negociación de aplicación exacta

La capa de aplicación reutiliza el sobre PR-07 sin duplicar campos en payload:

- `link.hello`: estación → Dinamizador, `kind=request`, solo después de mTLS;
  protocolo `2`, schema `1`; claims `source_id/station_id/center_id`; sin época,
  secuencia, `link_id` ni `idempotency_key`. Payload exacto con
  `build_version` no vacío y `capabilities` único/no vacío que incluye
  `secure_link_v1`.
- `link.accept`: Dinamizador → estación, `kind=response`, `correlation_id`
  exactamente igual al `message_id` del hello; claims ya vinculados;
  `connection_epoch` aleatorio no cero `uint64`; `link_id` no secreto opcional;
  sin secuencia ni `idempotency_key`. Payload exacto
  `negotiated_capabilities`, intersección allowlisted conocida e incluyendo
  `secure_link_v1`.
- `link.reject`: solo después de mTLS y de un hello estructuralmente utilizable y
  correlacionable; `kind=response`, correlación exacta; sin época, secuencia ni
  estado ACTIVE. Payload con exactamente una categoría aprobada.

Solo un `link.accept` completamente verificado mueve a `ACTIVE`. Protocolo `2` y
schema `1` son exactos; incompatibilidad no activa ni obtiene fallback
privilegiado. PR-08 no anuncia o habilita capacidad de negocio alguna.

## 8. Rechazos y canal de entrega

### Auditoría TLS/security local, sin frame JSON

Cuando TLS falla antes de tráfico de aplicación se audita localmente, según la
información segura disponible: `AUTH_REQUIRED`, `CERT_INVALID`, `CERT_EXPIRED` y
`CERT_REVOKED`. Certificado todavía no válido, CA no confiable y fallo TLS se
registran como subrazones estructuradas si la taxonomía local admite detalle. Un
TLS alert no es `link.reject` y no se inventa payload wire con secretos.

### Categorías wire elegibles después de mTLS

Según contexto y solo si existe correlación segura:

`IDENTITY_MISMATCH`, `CENTER_MISMATCH`, `PROTOCOL_UNSUPPORTED`,
`SCHEMA_UNSUPPORTED`, `CAPABILITY_REQUIRED`, `HANDSHAKE_MALFORMED`,
`HANDSHAKE_TIMEOUT`, `REPLAY`, `SEQUENCE_INVALID`, `PEER_REPLACED` y
`OUT_OF_SCOPE`.

Unsupported protocol/schema puede auditarse localmente y cerrar cuando todavía
no se ha establecido compatibilidad segura de respuesta. `HANDSHAKE_TIMEOUT`
solo usa wire si puede correlacionarse. Las categorías de tráfico ACTIVE se
registran y cierran sin inventar un nuevo tipo de mensaje. `REJECTED` describe
estado, no prueba entrega de un frame.

## 9. Época, secuencia, replay y tiempo

- `connection_epoch` lo genera Dinamizador por conexión autenticada exitosa:
  aleatorio criptográfico, no cero, `uint64`, no reutilizado ni persistido.
- Handshake y `link.accept` no llevan `sequence`; accept asigna la época.
- Todo frame v2 post-ACTIVE lleva época actual y secuencia direccional por peer,
  iniciada en `1` y con incremento exacto `+1`.
- Duplicado o valor menor es replay; valor mayor con gap se audita, cierra y
  exige reautenticación. No se salta silenciosamente.
- Reconexión asigna época nueva y reinicia secuencias. No hay wrap; se cierra
  antes de overflow de `MaxUint64`.
- `link_id`, si fue emitido, debe coincidir. El FIFO PR-07 de `message_id` sigue
  siendo dedupe de transporte separado.
- `sent_at` es dato observacional/auditable y no reloj primario de replay ni de
  certificado.
- X.509 `notBefore/notAfter` usa reloj de pared del OS local, sin dependencia de
  Internet/NTP. Un reloj local implausible/no confiable impide un nuevo ACTIVE.
  La conexión se cierra no después de `notAfter` del peer.
- Protección absoluta contra rollback de reloj es imposible totalmente offline
  sin fuente de tiempo confiable; la limitación queda registrada.

## 10. Legado y elegibilidad privilegiada

El parser legado puede permanecer, pero compatibilidad no equivale a
autorización. Una mutación privilegiada legada no autenticada nunca se ejecuta;
un fallo v2 no degrada a privilegios legados. Un adaptador futuro solo será
aceptable si ofrece controles equivalentes.

La elegibilidad futura requiere conjuntamente vínculo autenticado vigente,
identidad/centro válidos, versión exacta, capacidad obligatoria, época actual,
secuencia fresca, operación allowlisted y autorización posterior del workflow.
PR-08 habilita **cero acciones de producto**: no agrega handlers, migraciones,
`SessionActivation`, Coordinator, persistencia ni `ACTIVATE_SESSION`.

## 11. `SecurityAuditSink`

El sink es local y durable. Su allowlist contiene:

- `event_id`, `event_category`, `timestamp` y `result`;
- IDs confiables de peer/estación/centro cuando se conozcan;
- versiones de protocolo/schema;
- resumen de capacidades negociadas;
- `link_id` no secreto;
- categoría de rechazo.

Nunca contiene secretos, credenciales, claves privadas, material de clave
pública innecesario, PIN, verificadores, frame crudo, payload completo, payload
opaco de negocio, PII sensible ni datos de discapacidad. Fallar al auditar nunca
concede acceso. Si la auditoría obligatoria no está disponible, la elegibilidad
privilegiada sigue falsa; el vínculo puede permanecer técnicamente autenticado
solo para operación no privilegiada. Retención, acceso y exportación son
políticas de negocio separadas.

## 12. Tamaño y subdivisión obligatoria

Los forecasts umbrella originales son históricos y no sustituyen el estado
integrado ni la descomposición actual:

| Umbrella/slice | Tamaño | Estado autoritativo |
| --- | ---: | --- |
| PR-08A1 | 641 actual | CLOSED como A1a + A1b. |
| PR-08A2 | Hijos separados | CLOSED como A2a + A2b + A2c1 + A2c2. |
| PR-08B1 | Hijos separados | IN PROGRESS; umbrella no ejecutable. |
| PR-08B1a1 | 88 actual: 25 producción + 63 pruebas | CLOSED/INTEGRATED. |
| PR-08B1a2a1f | 166 líneas de fixtures | CLOSED/INTEGRATED/PUSHED; TEST-ONLY. |
| PR-08B1a2a1 | **299 actual: 102 producción + 197 pruebas** | CLOSED/INTEGRATED/PUSHED. |
| PR-08B1a2a1r | Sub-slices r1, r2a, r2a1 y r2b | CLOSED/INTEGRATED: Policy A (rechazo y auditoría de reanudación TLS). |
| PR-08B1a2a2 | a2a 179 neto + a2b 367 neto | CLOSED/INTEGRATED: a2a (primitivas y dependencias WSS) + a2b (integración WSS y lifecycle seguro); cada hijo ≤400 neto. |
| PR-08B1a2b | Preflight fresco ≤400 | NEXT / PREFLIGHT REQUIRED / NOT STARTED: resource controls/hardening separado. |
| PR-08B1b | Planificación futura | PLANNED / BLOCKED por identidad X.509. |
| PR-08B1c | Planificación futura | PLANNED / NOT STARTED. |
| PR-08B2 | 170–220 forecast histórico | Depende de B1 completo. |
| PR-08U1 | 250–330 forecast | Tras freeze A1/A2 y aceptación del spike de storage. |
| PR-08U2 | 230–300 forecast | Depende de U1. |

Cada slice ejecutable tiene target máximo ≤400. No se aprueba excepción
401–450. Los umbrellas son no ejecutables, no poseen diff propio y no deben
ocultar los tamaños reales de sus hijos.

## 13. Orden y readiness

```text
Dinamizador:
PR-02 integrado → A1 CLOSED → A2 CLOSED → B1a1 CLOSED → B1a2a1f CLOSED → B1a2a1 CLOSED → B1a2a1r CLOSED → B1a2a2 CLOSED → B1a2b NEXT/PREFLIGHT REQUIRED → B1b BLOCKED → B1c → B2 → PR-09 → PR-10 → PR-11

Usuario PC:
PR-07B COMPLETE/integrado → freeze contractual PR-08A1/A2
                            → spike storage aceptado → PR-08U1 → PR-08U2

Cross-repo:
PR-08U1 puede avanzar en paralelo con B después del freeze A1/A2.
PR-11 depende de PR-09/PR-10 + A/B/U conformes e integrados.
```

PR-09 y PR-10 permanecen inertes/de capa de sesión según su alcance; PR-08 no
adelanta sus handlers. PR-08U tiene exactamente las responsabilidades aprobadas
y no añade handlers de producto, migraciones ni conducta de sesión. Antes de
PR-11 se exige conformidad cross-repo.

**Readiness:** los nueve bloqueadores arquitectónicos originales permanecen
cerrados, A1/A2 están cerrados y B1 está en progreso. No se crean worktrees
umbrella PR-08A, PR-08B, PR-08B1 o PR-08U. La próxima slice técnica es
B1a2b (control de recursos/hardening de transporte), solo tras un preflight
fresco de arquitectura, alcance y tamaño ≤400; no está lista para implementar.
Las slices de transporte, integradas o posteriores, no autorizan identidad,
ACTIVE ni acciones de producto.

## 14. Riesgos y limitaciones aceptadas

- Revocación global inmediata es imposible durante operación totalmente offline;
  deny/disabled local conocido permanece fail-closed.
- Rollback absoluto del reloj no puede detectarse offline sin tiempo confiable;
  reloj local implausible impide nuevas conexiones ACTIVE.
- El spike de storage todavía debe seleccionar adaptador y demostrar la
  no salida/exportación/copia de claves privadas.
- La política TLS resumption está resuelta (Policy A, CLOSED / INTEGRATED): se
  rechazan las conexiones reanudadas. `SSL_OP_NO_TICKET` por sí solo no
  constituye prueba de no-reanudación bajo Node 22/OpenSSL 3; se conserva como
  defensa en profundidad.
- La codificación X.509 de identidad de negocio y la provisión física del
  certificado/clave de servidor Dinamizador permanecen OPEN; no se inventan.
- La plausibilidad del reloj local permanece OPEN aunque existan primitivas de
  validación temporal.
- Retención, acceso y exportación de auditoría siguen siendo política de negocio;
  no se inventan en esta arquitectura.
- Cualquier expansión a capacidades o acciones de producto requiere contrato y
  autorización posteriores; `secure_link_v1` por sí sola no concede acciones.
