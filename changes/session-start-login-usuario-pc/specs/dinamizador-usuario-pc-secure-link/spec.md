# Especificación normativa del vínculo seguro Dinamizador ↔ Usuario PC

## Purpose

Definir el contrato cross-component de PR-08 para autenticar instalaciones,
negociar Agent v2 y proteger tráfico LAN privilegiado de manera offline-first,
sin habilitar acciones de producto. Este contrato aplica conjuntamente a
Soft_Dinamizador y Soft_Usuario_PC.

## Requirements

### Requirement: Transporte WSS/TLS 1.3 con autenticación mutua

Todo tráfico LAN confiable que aspire a elegibilidad privilegiada MUST usar WSS
sobre TLS 1.3 con mTLS. Usuario PC MUST actuar como cliente WSS y Dinamizador
como servidor WSS. El sistema MUST NOT aceptar WS en claro, contraseña global o
compartida, ni handshake criptográfico propio como sustituto. TLS 1.3 MUST
proporcionar la prueba mutua y la protección del transcript; early data/0-RTT
MUST estar deshabilitado.

#### Scenario: Conexión mTLS satisfactoria

- GIVEN una estación y un Dinamizador con certificados vigentes y localmente confiables
- AND ambos poseen registros de instalación autorizados para el mismo centro
- WHEN Usuario PC establece TLS 1.3 mutuo y completa el upgrade WSS
- THEN el vínculo MUST entrar en `AUTHENTICATED_NEGOTIATING`
- AND MUST NOT estar todavía en `ACTIVE` hasta validar registro, centro y negociación

#### Scenario: WS en claro o secreto compartido

- GIVEN un peer que ofrece WS en claro, una contraseña compartida o un handshake criptográfico custom
- WHEN intenta obtener confianza para tráfico privilegiado
- THEN el sistema MUST rechazar la ruta
- AND MUST otorgar cero elegibilidad privilegiada

### Requirement: PKI privada, identidad por instalación y trust local

Una CA privada controlada por Infoplazas MUST emitir certificados por
instalación. Su clave privada firmante MUST NOT residir en endpoints. Estación y
Dinamizador MUST usar identidades/certificados únicos, nunca compartidos entre
instalaciones, estaciones o centros. Una identidad confiable MUST exigir
certificado válido, registro de instalación autorizado y binding exacto al
centro. La identidad humana del operador MUST permanecer separada.

`station_id`, `center_id`, `source_id` y `link_id` del sobre MUST permanecer
claims hasta comprobarlos contra certificado, registro autorizado y binding de
centro. IP, MAC, hostname, nombre visible o mDNS MUST NOT conferir identidad.

#### Scenario: Identidad y centro coinciden

- GIVEN un certificado de estación válido vinculado a una instalación autorizada
- AND el registro local vincula esa instalación al centro esperado
- WHEN Dinamizador compara los claims del hello con certificado y registro
- THEN MUST reconocer la identidad de instalación y centro vinculados
- AND MAY continuar la negociación sin conferir todavía acciones de producto

#### Scenario: Mismatch de identidad

- GIVEN mTLS completado
- BUT `source_id` o `station_id` no coincide con la identidad registrada del certificado
- WHEN Dinamizador valida los claims
- THEN MUST impedir `ACTIVE`
- AND MAY emitir `link.reject` con `IDENTITY_MISMATCH` si el hello es correlacionable
- AND MUST auditar únicamente identificadores confiables cuando se conozcan

#### Scenario: Mismatch de centro

- GIVEN mTLS completado con una instalación conocida
- BUT el `center_id` declarado o el peer registrado no corresponde al centro autorizado
- WHEN el sistema valida el binding
- THEN MUST impedir `ACTIVE`
- AND MAY emitir `link.reject` con `CENTER_MISMATCH` si existe correlación segura

### Requirement: Enrolamiento y almacenamiento protegido

Cada endpoint MUST generar su clave privada localmente y usar CSR. La emisión
MUST requerir autorización mediante flujo online o paquete offline firmado. Todo paquete offline MUST estar vinculado exactamente al CSR y nonce actuales, y MUST avanzar un estado de autorización monotónico (high-water mark) para prevenir replay y rollback. La
raíz/CA pública MUST fijarse localmente para confianza runtime offline.

En Windows, las claves privadas MUST usar almacenamiento protegido y ACL
restrictiva. Un spike MUST seleccionar Certificate Store/CNG cuando sea práctico
o almacenamiento de aplicación protegido con DPAPI. El adaptador seleccionado
MUST demostrar que la clave privada nunca sale de la instalación y que no existe
ruta de texto plano, exportación o copia. Si DPAPI no demuestra protección
equivalente, MUST fallar el spike. Esta especificación MUST NOT seleccionar el
adaptador antes del spike.

#### Scenario: Provisionamiento offline autorizado

- GIVEN una instalación sin acceso a Internet
- AND un paquete de provisionamiento firmado por la autoridad autorizada
- WHEN la instalación presenta su CSR y aplica el paquete válido
- THEN MUST instalar solo su certificado y trust público local
- AND la clave firmante de CA MUST permanecer fuera del endpoint
- AND la clave privada de instalación MUST permanecer en el dispositivo

#### Scenario: Adaptador de storage no demuestra no exportación

- GIVEN un candidato de almacenamiento protegido
- WHEN el spike encuentra una ruta de texto plano, exportación o copia de la clave privada
- THEN el candidato MUST ser rechazado
- AND MUST NOT convertirse en decisión por conveniencia de implementación

### Requirement: Operación runtime completamente offline

La autenticación runtime MUST funcionar sin Internet ni NTP cuando ambos peers
tengan credenciales locales no expiradas, raíz pública local confiable, registros
de autorización y estado local conocido. La nube MUST NOT participar en cada
conexión.

#### Scenario: LAN disponible sin Internet

- GIVEN estación y Dinamizador aislados de Internet
- AND sus certificados están vigentes según el reloj OS local confiable
- AND los registros locales los autorizan para el mismo centro
- WHEN establecen mTLS y completan la negociación
- THEN MUST poder alcanzar `ACTIVE`
- AND MUST NOT requerir consulta central online

### Requirement: Ciclo de vida, reemplazo y revocación

Los certificados MUST tener vigencia finita. Renovación/reemisión de certificado
MUST distinguirse de rotación de clave privada/nuevo enrolamiento. Reinstalar o
reemplazar una instalación MUST generar clave y credencial nuevas y MUST
deshabilitar/revocar la anterior.

Un overlap temporal MAY existir solo para una instalación bajo cutover explícito;
la credencial anterior MUST negarse al completar cutover o al expirar. Ambos
endpoints MUST conservar peers autorizados/deshabilitados; Dinamizador MUST tener
un registro local deny/de estaciones deshabilitadas. Estado local conocido como
inválido, revocado o deshabilitado MUST fallar cerrado. La revocación central
MUST sincronizarse cuando sea posible. El sistema MUST documentar que revocación
global inmediata es imposible durante aislamiento totalmente offline. PR-08
MUST NOT fijar una cadencia de rotación.

#### Scenario: Reinstalación o reemplazo

- GIVEN una estación o instalación Dinamizador reinstalada o reemplazada
- WHEN vuelve a enrolarse
- THEN MUST generar una clave y credencial nuevas
- AND MUST deshabilitar o revocar la credencial anterior
- AND la conexión anterior MUST perder elegibilidad inmediatamente cuando el cambio sea localmente conocido

#### Scenario: Estación en deny local

- GIVEN una estación cuyo registro local está revocado o deshabilitado
- WHEN presenta un certificado criptográficamente válido
- THEN Dinamizador MUST impedir `ACTIVE`
- AND MUST mantener el rechazo aunque no haya Internet

#### Scenario: Revocación global durante aislamiento

- GIVEN una revocación central todavía no sincronizada por aislamiento total
- WHEN un endpoint solo dispone del último estado local conocido
- THEN MUST aplicar ese estado local sin afirmar revocación global inmediata
- AND MUST sincronizar la revocación central al recuperar conectividad

### Requirement: FSM del vínculo y reanudación segura

La FSM MUST seguir:

```text
DISCONNECTED
  → CONNECTED_UNAUTHENTICATED
  → AUTHENTICATING
  → AUTHENTICATED_NEGOTIATING
  → ACTIVE

estado aplicable → REJECTED o CLOSING → DISCONNECTED
```

`CONNECTED_UNAUTHENTICATED` MUST representar solo TCP. `AUTHENTICATING` MUST
abarcar TLS 1.3 mTLS y WebSocket upgrade. Cada nueva conexión/reconexión MUST
empezar sin autenticación, época, secuencia, capacidades ni privilegios
heredados. Dinamizador MUST exponer a lo sumo un enlace ACTIVE por station_id validado. El reemplazo del peer MUST invalidar el binding anterior atómicamente.

La reanudación TLS MAY usarse solo si cada conexión nueva revalida vigencia del
certificado, identidad autenticada, registro de autorización, binding de centro y
deny local. Si el stack no puede garantizarlo, MUST deshabilitarse.

#### Scenario: Reconexión limpia

- GIVEN una conexión previamente `ACTIVE`
- WHEN se desconecta y vuelve a conectar
- THEN MUST empezar sin auth, época, secuencias ni capacidades heredadas
- AND MUST repetir mTLS, autorización local y negociación

#### Scenario: Reanudación sin revalidación completa

- GIVEN un stack de TLS resumption que no revalida todos los controles actuales
- WHEN se configura el gateway o cliente
- THEN la reanudación MUST quedar deshabilitada
- AND la conexión MUST realizar autenticación completa

### Requirement: `link.hello` exacto

Después de mTLS, Usuario PC MUST enviar un sobre PR-07 con:

- `protocol_version=2`, `payload_schema_version=1`;
- `message_id` válido, `kind=request`, `type=link.hello` y `sent_at`;
- claims `source_id`, `station_id` y `center_id`;
- ausencia de `connection_epoch`, `sequence`, `link_id` e `idempotency_key`;
- payload exacto `{build_version: string no vacío, capabilities: string[]}`,
  donde `capabilities` sea no vacío, sin duplicados e incluya `secure_link_v1`.

El payload MUST NOT duplicar campos del sobre ni contener desafío custom.

#### Scenario: Hello válido después de mTLS

- GIVEN mTLS completado y claims disponibles
- WHEN Usuario PC emite `link.hello`
- THEN el frame MUST satisfacer el schema mínimo exacto
- AND MUST anunciar `secure_link_v1`
- AND MUST permanecer sin época ni secuencia

#### Scenario: Hello malformado o sin capacidad obligatoria

- GIVEN un hello correlacionable con campos extra prohibidos, payload inválido o sin `secure_link_v1`
- WHEN Dinamizador lo valida
- THEN MUST impedir `ACTIVE`
- AND MAY responder respectivamente `HANDSHAKE_MALFORMED` o `CAPABILITY_REQUIRED`

### Requirement: `link.accept` exacto y negociación allowlisted

Dinamizador MUST responder un hello válido con un sobre PR-07 que incluya:

- `protocol_version=2`, `payload_schema_version=1`, `message_id`, `sent_at`;
- `kind=response`, `type=link.accept` y `correlation_id` exactamente igual al
  `message_id` del hello;
- claims `source_id`, `station_id` y `center_id` ya vinculados;
- `connection_epoch` criptográficamente aleatorio, no cero y `uint64`;
- `link_id` no secreto opcional;
- ausencia de `sequence` e `idempotency_key`;
- payload exacto `{negotiated_capabilities: string[]}` sin duplicados, igual a la
  intersección de capacidades conocidas allowlisted e incluyendo
  `secure_link_v1`.

Solo un accept completamente verificado MUST mover Usuario PC a `ACTIVE`.
Capacidades desconocidas MUST conceder cero privilegio. PR-08 MUST anunciar y
habilitar cero capacidades de negocio.

#### Scenario: Intersección allowlisted

- GIVEN un hello que anuncia capacidades conocidas y desconocidas
- WHEN Dinamizador negocia
- THEN `negotiated_capabilities` MUST contener exactamente la intersección conocida allowlisted
- AND MUST incluir `secure_link_v1`
- AND las capacidades desconocidas MUST otorgar cero privilegio

#### Scenario: Accept no verificable

- GIVEN un `link.accept` con correlación incorrecta, claims no vinculados, época cero o capacidad no negociada
- WHEN Usuario PC lo valida
- THEN MUST impedir la transición a `ACTIVE`
- AND MUST cerrar o rechazar de forma fail-closed

### Requirement: `link.reject` y separación de canal TLS

`link.reject` MUST usarse solo después de mTLS y ante un hello estructuralmente
utilizable y correlacionable. MUST ser `kind=response`, usar
`type=link.reject`, correlacionar exactamente el hello, carecer de época,
secuencia y estado ACTIVE, y contener exactamente una categoría aprobada.

Cuando TLS falla antes del tráfico de aplicación, el sistema MUST auditar
localmente según corresponda `AUTH_REQUIRED`, `CERT_INVALID`, `CERT_EXPIRED` o
`CERT_REVOKED`; certificado todavía no válido, CA no confiable y fallo TLS MAY
ser subrazones estructuradas si existe taxonomy de detalle. Un TLS alert MUST NOT
considerarse `link.reject` y MUST NOT inventarse un payload JSON secreto.

Después de mTLS, las categorías wire elegibles según contexto son
`IDENTITY_MISMATCH`, `CENTER_MISMATCH`, `PROTOCOL_UNSUPPORTED`,
`SCHEMA_UNSUPPORTED`, `CAPABILITY_REQUIRED`, `HANDSHAKE_MALFORMED`,
`HANDSHAKE_TIMEOUT`, `REPLAY`, `SEQUENCE_INVALID`, `PEER_REPLACED` y
`OUT_OF_SCOPE`. `HANDSHAKE_TIMEOUT` solo MAY enviarse si puede correlacionarse.
Protocolo/schema incompatible MAY auditarse localmente y cerrar cuando no exista
compatibilidad segura para responder. Estado `REJECTED` MUST NOT implicar que se
envió un frame.

#### Scenario: TLS falla antes de aplicación

- GIVEN un certificado expirado durante el handshake TLS
- WHEN mTLS falla antes de aceptar tráfico de aplicación
- THEN el endpoint MUST auditar localmente `CERT_EXPIRED`
- AND MUST NOT enviar `link.reject`
- AND MAY emitir únicamente el TLS alert estándar aplicable

#### Scenario: Reject correlacionable después de mTLS

- GIVEN mTLS exitoso y un hello estructuralmente utilizable con centro incorrecto
- WHEN Dinamizador rechaza la negociación
- THEN MAY enviar `link.reject` con `CENTER_MISMATCH`
- AND `correlation_id` MUST coincidir exactamente con el hello
- AND el vínculo MUST NOT entrar en `ACTIVE`

#### Scenario: Protocolo incompatible sin respuesta segura

- GIVEN un candidato con protocolo o schema incompatible
- AND no está establecida compatibilidad segura para un response
- WHEN el gateway lo procesa
- THEN MUST auditar localmente el rechazo compatible con su taxonomy
- AND MUST cerrar sin downgrade ni JSON inventado

### Requirement: Versión exacta y no downgrade

La negociación MUST aceptar únicamente `protocol_version=2` y
`payload_schema_version=1`. Un peer incompatible MUST NOT alcanzar `ACTIVE`, usar
v1 como fallback privilegiado ni activar una ruta insegura. El parser legado MAY
permanecer, pero una mutación privilegiada legada no autenticada MUST NEVER
ejecutarse. Un fallo v2 MUST NOT intentar autorización legada. Un adaptador
legado futuro MUST demostrar controles equivalentes.

#### Scenario: Intento de downgrade

- GIVEN un peer que falla negociación v2 y luego presenta un comando legado privilegiado
- WHEN el sistema clasifica el tráfico
- THEN MUST negar la mutación
- AND MUST NOT convertir el fallo v2 en autorización v1

### Requirement: Época de conexión

Por cada conexión autenticada y negociada con éxito, Dinamizador MUST generar un
`connection_epoch` criptográficamente aleatorio, no cero y representable como
`uint64`. MUST ser nuevo por conexión, no reutilizado y no persistido. Un
reconnect MUST recibir época nueva; tráfico con época anterior MUST ser stale o
replay.

#### Scenario: Reconexión asigna época nueva

- GIVEN una conexión ACTIVE con una época previa
- WHEN el mismo peer reconecta y completa otra negociación
- THEN Dinamizador MUST asignar una época distinta y no cero
- AND todo frame de la época anterior MUST ser rechazado

### Requirement: Secuencia direccional y overflow

Handshake, `link.hello`, `link.accept` y `link.reject` MUST carecer de secuencia.
Todo frame v2 post-ACTIVE MUST incluir la época actual y una secuencia `uint64`
direccional por peer. Cada dirección MUST comenzar en `1` y aceptar solo el
siguiente valor exacto `+1`. Duplicado o menor MUST clasificarse replay; un valor
mayor con gap MUST auditarse, cerrar y exigir reautenticación. El sistema MUST
NOT saltar silenciosamente. Reconnect MUST reiniciar secuencias bajo época nueva.
El sistema MUST cerrar antes de overflow de `MaxUint64` y MUST NOT hacer wrap. Si
se emitió `link_id`, cada frame post-ACTIVE MUST coincidir. El FIFO PR-07 por
`message_id` MUST permanecer como dedupe de transporte separado.

#### Scenario: Secuencia exacta

- GIVEN un vínculo ACTIVE cuya siguiente secuencia entrante es 7
- WHEN llega un frame con época actual y secuencia 7
- THEN MAY pasar al siguiente guard
- AND la siguiente secuencia esperada MUST ser 8

#### Scenario: Duplicado, valor menor o gap

- GIVEN un vínculo ACTIVE con siguiente secuencia esperada 7
- WHEN llega secuencia 6 o un duplicado
- THEN MUST rechazarse como replay
- BUT WHEN llega secuencia 8 o mayor
- THEN MUST auditar `SEQUENCE_INVALID`, cerrar y exigir reautenticación
- AND MUST NOT saltar el valor faltante

#### Scenario: Límite uint64

- GIVEN una dirección próxima a `MaxUint64`
- WHEN no puede emitir otro valor sin wrap
- THEN MUST cerrar el vínculo antes del overflow
- AND MUST requerir conexión y época nuevas

### Requirement: Tiempo, expiración y limitación offline

`sent_at` MUST permanecer obligatorio y observacional para auditoría; MUST NOT
ser la defensa primaria de replay ni el reloj de certificado. X.509
`notBefore/notAfter` MUST validarse con reloj de pared del OS local y MUST NOT
depender de Internet/NTP. Un reloj local implausible o no confiable MUST impedir
una nueva transición a `ACTIVE`; MUST NOT omitir validación temporal. Una
conexión MUST cerrarse no después de `notAfter` del certificado peer. La
arquitectura MUST registrar que protección absoluta contra rollback del reloj es
imposible totalmente offline sin tiempo confiable.

#### Scenario: Certificado expira durante conexión

- GIVEN un vínculo autenticado cuyo certificado peer se aproxima a `notAfter`
- WHEN alcanza su expiración
- THEN el sistema MUST cerrar la conexión no después de ese instante
- AND MUST exigir credencial vigente para reconectar

#### Scenario: Reloj local implausible

- GIVEN que el endpoint no puede confiar razonablemente en su reloj OS local
- WHEN intenta validar `notBefore/notAfter` para una conexión nueva
- THEN MUST impedir `ACTIVE`
- AND MUST NOT ignorar la vigencia por estar offline

### Requirement: Guardia privilegiada y cero acciones de producto

Autenticación MUST ser necesaria y no suficiente. Toda elegibilidad futura MUST
exigir vínculo autenticado actual, identidad/centro válidos, versión exacta,
`secure_link_v1`, época actual, secuencia fresca, operación allowlisted y
autorización posterior del workflow. PR-08 MUST habilitar cero handlers o
acciones de producto y MUST NOT agregar migraciones, `SessionActivation`,
Coordinator, persistencia de sesión ni `ACTIVATE_SESSION`.

#### Scenario: Vínculo ACTIVE sin handler de producto

- GIVEN un vínculo mTLS y negociación válidos en `ACTIVE`
- WHEN no existe un handler de producto aprobado posteriormente
- THEN el sistema MUST ejecutar cero mutaciones de producto
- AND `secure_link_v1` MUST NOT interpretarse como permiso de sesión o estación

### Requirement: `SecurityAuditSink` durable y minimizado

Cada endpoint MUST disponer de un `SecurityAuditSink` local durable con allowlist:
`event_id`, `event_category`, `timestamp`, `result`, IDs confiables de
peer/estación/centro cuando se conozcan, versiones de protocolo/schema, resumen
de capacidades negociadas, `link_id` no secreto y categoría de rechazo. MUST NOT
existir bags arbitrarios de metadata o contexto.

La taxonomía de `event_category` es exactamente `AUTHENTICATION`,
`IDENTITY_BINDING`, `NEGOTIATION`, `REPLAY_PROTECTION`, `LINK_LIFECYCLE` y
`PRIVILEGE_ELIGIBILITY`. El `result` es exactamente `SUCCESS` cuando el control
se completa, `REJECTED` cuando se niega de forma controlada, `INVALIDATED` cuando
se invalida un vínculo existente, o `FAILED` ante fallo técnico. La entrega al
sink es independiente del `result`: exactamente `RECORDED` cuando el evento se
persiste o `UNAVAILABLE` cuando el sink no está disponible.

| `event_category` | códigos canónicos de rechazo |
| --- | --- |
| `AUTHENTICATION` | `AUTH_REQUIRED`, `CERT_INVALID`, `CERT_EXPIRED`, `CERT_REVOKED` |
| `IDENTITY_BINDING` | `IDENTITY_MISMATCH`, `CENTER_MISMATCH` |
| `NEGOTIATION` | `PROTOCOL_UNSUPPORTED`, `SCHEMA_UNSUPPORTED`, `CAPABILITY_REQUIRED`, `HANDSHAKE_MALFORMED`, `HANDSHAKE_TIMEOUT` |
| `REPLAY_PROTECTION` | `REPLAY`, `SEQUENCE_INVALID` |
| `LINK_LIFECYCLE` | `PEER_REPLACED` |
| `PRIVILEGE_ELIGIBILITY` | `OUT_OF_SCOPE` |

`REJECTED` MUST llevar un código canónico de rechazo; `INVALIDATED` actualmente
MUST llevar `PEER_REPLACED`; `SUCCESS` y `FAILED` MUST NOT llevar código de
rechazo. Las combinaciones contradictorias están prohibidas. `UNAVAILABLE` MUST
NOT conceder privilegio ni transformar un rechazo en aceptación; un vínculo
técnicamente autenticado MAY permanecer no privilegiado.

El sink MUST ser durable pero acotado, rotar atómicamente e implementar rate control/coalescing ante eventos de alta frecuencia (fallos TLS, gaps) para prevenir agotamiento de disco. El sink MUST NOT guardar secretos, credenciales, claves privadas, material de
clave pública innecesario, PIN, verificadores, frames crudos, payload completo,
payload opaco de negocio, PII sensible ni datos de discapacidad. Fallar al
auditar MUST NOT conceder privilegio. Si auditoría obligatoria no está
disponible, elegibilidad privilegiada MUST permanecer falsa; el vínculo MAY
seguir técnicamente autenticado para operación no privilegiada. Retención,
acceso y exportación permanecen política de negocio separada.

#### Scenario: Fallo del sink obligatorio

- GIVEN mTLS y negociación criptográfica válidos
- BUT el sink durable obligatorio no está disponible
- WHEN se evalúa elegibilidad privilegiada
- THEN MUST permanecer falsa
- AND el vínculo MAY conservar solo operación técnicamente autenticada no privilegiada
- AND el fallo MUST NOT transformarse en aceptación

#### Scenario: Redacción de auditoría

- GIVEN un frame con payload o dato sensible
- WHEN se registra un evento de seguridad
- THEN el sink MUST persistir solo campos de la allowlist
- AND MUST omitir frame/payload crudo, secretos, PII sensible y datos de discapacidad

### Requirement: Responsabilidad y conformidad de PR-08U

Después del freeze contractual PR-08A1/A2, Usuario PC MUST implementar únicamente
la contraparte aprobada: credencial protegida; cliente WSS/mTLS; verificación del
Dinamizador; binding de peer; negociación/FSM; época/secuencia/reconexión; manejo
de rechazo y auditoría. PR-08U MUST dividirse en PR-08U1 y PR-08U2 y MUST NOT
agregar handlers de producto, migraciones ni conducta de sesión. La conformidad
cross-repo A+B+U MUST verificarse antes de PR-11. PR-08U1 MAY comenzar en
paralelo con B después del freeze A1/A2 y de aceptar el spike de almacenamiento.

#### Scenario: Conformidad cross-repo previa a PR-11

- GIVEN A1/A2, B1/B2 y U1/U2 implementados en sus repositorios independientes
- WHEN se planifica PR-11
- THEN MUST existir evidencia de conformidad de mTLS, negociación, rechazo, época/secuencia y auditoría
- AND PR-11 MUST permanecer bloqueado si esa evidencia falta

#### Scenario: PR-08U no adelanta sesión

- GIVEN una implementación conforme de PR-08U1/U2
- WHEN se inspecciona su alcance
- THEN MUST contener cero handlers, migraciones o activaciones de sesión de producto
- AND MUST limitarse al vínculo seguro y su auditoría
