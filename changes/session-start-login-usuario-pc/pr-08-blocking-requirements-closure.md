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
4. Ejecutar un preflight nuevo antes del primer worktree potencial: PR-08A1.

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
lo garantiza, se deshabilita la reanudación.

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

La estimación post-mTLS de Sol sustituye todos los forecasts provisionales anteriores:

| Umbrella/slice | Producción | Pruebas | Total | Estado |
| --- | ---: | ---: | ---: | --- |
| PR-08A | 180–230 | 220–280 | 400–510 | Umbrella no ejecutable; `>450` plausible. Superseded por A1+A2. |
| PR-08A1 — contrato/schema/FSM/capacidad/versión | 85–110 | 105–140 | 190–250 | Ejecutable solo tras preflight nuevo. |
| PR-08A2 — época/secuencia/replay/guard privilegiado | 95–120 | 115–140 | 210–260 | Ejecutable después de A1. |
| PR-08B | 170–230 | 210–280 | 380–510 | Umbrella no ejecutable; `>450` plausible. Superseded por B1+B2. |
| PR-08B1 — WSS/TLS, identidad/centro, lifecycle/clock/resumption | 95–130 | 115–160 | 210–290 | Depende de contratos A verificados e integrados. |
| PR-08B2 — auditoría durable + composición Electron | 75–100 | 95–120 | 170–220 | Depende de B1. |
| PR-08U | 220–290 | 260–340 | 480–630 | Umbrella no ejecutable; split obligatorio. Superseded por U1+U2. |
| PR-08U1 — credencial protegida + cliente WSS/mTLS + peer binding | 115–150 | 135–180 | 250–330 | Tras freeze A1/A2 y aceptación del spike de storage. |
| PR-08U2 — negociación/FSM/época-secuencia/reconnect/audit | 105–140 | 125–160 | 230–300 | Depende de U1. |

Reconciliación exacta:

- A1+A2 = 180–230 producción + 220–280 pruebas = **400–510**.
- B1+B2 = 170–230 producción + 210–280 pruebas = **380–510**.
- U1+U2 = 220–290 producción + 260–340 pruebas = **480–630**.
- Dinamizador A+B = **780–1,020**.
- Total cross-repo A+B+U = **1,260–1,650**.

Cada slice ejecutable tiene target máximo ≤400. No se aprueba excepción
401–450. Los umbrellas quedan no ejecutables/superseded por subdivisión, no
completados.

## 13. Orden y readiness

```text
Dinamizador:
PR-02 integrado → PR-08A1 → PR-08A2 → PR-08B1 → PR-08B2 → PR-09 → PR-10 → PR-11

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

**Readiness:** los nueve requisitos originales están cerrados y listos para
planificación de implementación. No están listos para crear worktrees umbrella
PR-08A, PR-08B o PR-08U. No se crea worktree en este cierre. El primer worktree
potencial es PR-08A1, únicamente después de un preflight de implementación
nuevo.

## 14. Riesgos y limitaciones aceptadas

- Revocación global inmediata es imposible durante operación totalmente offline;
  deny/disabled local conocido permanece fail-closed.
- Rollback absoluto del reloj no puede detectarse offline sin tiempo confiable;
  reloj local implausible impide nuevas conexiones ACTIVE.
- El spike de storage todavía debe seleccionar adaptador y demostrar la
  no salida/exportación/copia de claves privadas.
- Retención, acceso y exportación de auditoría siguen siendo política de negocio;
  no se inventan en esta arquitectura.
- Cualquier expansión a capacidades o acciones de producto requiere contrato y
  autorización posteriores; `secure_link_v1` por sí sola no concede acciones.
