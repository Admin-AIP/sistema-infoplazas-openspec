# Diseño técnico: inicio de sesión/login de Usuario PC

- **Cambio:** `session-start-login-usuario-pc`
- **Fase:** diseño y cierre de arquitectura solamente; no autoriza cambios de producción, `apply` ni worktrees.
- **Alcance principal:** `Soft_Usuario_PC/agente-infoplaza`, con extensiones necesarias en `Soft_Dinamizador` y contratos preparados para la plataforma central futura.
- **Fuentes:** propuesta y especificación aprobadas del cambio, contexto OpenSpec y guardrails de integración.

## 1. Resumen de la solución

El flujo se implementará como un caso de uso de sesión coordinado, no como un `UNLOCK` enriquecido. Usuario PC obtendrá una base SQLite embebida para conservar la proyección local mínima de usuarios y catálogos, credenciales protegidas, sesiones, auditoría y hechos pendientes. Una sesión solo podrá activar el mecanismo físico de desbloqueo después de confirmar atómicamente su núcleo durable: `session_id`, usuario, estación, modalidad, motivos, tiempos iniciales, estado y una intención/marca de sincronización. El hecho completo, el outbox expandido y la auditoría de negocio recuperable pueden escribirse en la misma transacción si no agregan latencia significativa, pero no son prerrequisitos de desbloqueo: un fallo secundario aislable se difiere mediante la marca y no revierte el núcleo; solo un fallo de durabilidad que también invalide el commit nuclear mantiene el bloqueo. Esta regla no relaja `SecurityAuditSink`: cuando la auditoría de seguridad del vínculo sea obligatoria, su indisponibilidad mantiene falsa toda elegibilidad privilegiada.

Soft_Dinamizador seguirá siendo la autoridad operativa de la Infoplaza y extenderá su modelo SQLite de `sesiones`; no se creará una segunda entidad de sesión. Su proceso principal Electron alojará el gateway LAN, los servicios de autorización/conciliación y los repositorios. El renderer seguirá accediendo únicamente por IPC. La plataforma central futura será la autoridad nacional de usuarios, catálogos y conflictos entre centros, pero no se diseña su Web UI ni sus reglas detalladas de roles/reportes.

El WebSocket actual evolucionará mediante un sobre versionado de solicitud/respuesta/evento con correlación e idempotencia. Los comandos `LOCK`, `UNLOCK`, `SET_TIMER`, `MESSAGE`, `SHUTDOWN` y `CANCEL_MAINTENANCE` se conservarán durante la transición, pero todo efecto privilegiado exigirá vínculo autenticado y defensas anti-replay/duplicado. Un handshake de capacidades evitará que una estación nueva interprete un comando legado sin contexto como aprobación de un login v2.

## 2. Evidencia del estado actual

### Usuario PC

- `internal/lockdown/fsm.go`, `app_lock.go` y los hooks existentes ya concentran el control físico de bloqueo, desbloqueo y mantenimiento.
- `internal/connection/protocol.go` solo acepta comandos server-push con `{type,payload}`.
- `internal/connection/client.go` ya puede leer y enviar por WebSocket, reconectar y publicar estado, pero la conexión actual es `ws://` y no autentica al servidor.
- `internal/commands/dispatcher.go` mantiene el temporizador únicamente en memoria y procesa `UNLOCK` sin exigir una sesión durable.
- `frontend/src/components/LockScreen.tsx` es pasivo; `TimerTab.tsx` y `PanelShell` ofrecen superficies reutilizables, aunque contienen datos y umbrales de demostración.
- `NewApp()` inicia la FSM en `unlocked`; no existe recuperación de sesión antes de elegir el estado inicial.
- `docs/Database_InfoplazaAgente.md` documenta que no hay base de datos porque el agente histórico solo tenía configuración y logs. Esa premisa deja de ser válida para relaciones, consultas, unicidad, outbox y commits atómicos exigidos por esta especificación.

### Dinamizador

- `electron/database/schema.ts` ya inicializa SQLite con migraciones, WAL y claves foráneas.
- `pcs.repository.ts` contiene el modelo existente de `pcs` y `sesiones`, genera UUID y agrega elementos a `sync_queue`.
- `sesionesRepository.iniciar()` valida una sesión activa del usuario, pero no tiene modalidad, procedencia, correlación, motivos ni conciliación; la creación y el alta de `sync_queue` no están en una única transacción.
- El cronómetro se proyecta en `pcs.store.ts` y persiste periódicamente desde el renderer; no debe ser la fuente durable de verdad del nuevo flujo.
- `AsignarUsuarioModal` y `PCsPage` preservan el flujo manual útil.
- No existe un servidor LAN/WebSocket de producción en el código fuente de Soft_Dinamizador ni una dependencia para él. La superficie `sync` aparece en preload/documentación, pero no tiene handlers correspondientes en `handlers.ts`.
- `packages/shared-types` ya define IDs UUID, `Sesion`, `PC`, `Usuario` y `SyncQueueItem`, pero el contrato de sesión es insuficiente y la resolución genérica “dashboard siempre gana” no es válida para hechos inmutables de sesión/auditoría.

## 3. Límites y responsabilidades

| Componente | Responsabilidad propia | No debe hacer |
| --- | --- | --- |
| **Usuario PC — React/Wails UI** | Capturar documento/PIN, motivos y opciones; mostrar espera, rechazo, degradación y widget mínimo; borrar el PIN del estado de UI al enviarlo. | Acceder a SQLite, decidir autoridad, registrar logs con credenciales o desbloquear directamente. |
| **Usuario PC — servicios Go** | Buscar identidad local, validar PIN y elegibilidad, aplicar política local, coordinar autorización, crear sesión durable, restaurar temporizador, controlar FSM, outbox/inbox y auditoría. | Usar documento/MAC/nombre como PK, confiar en el renderer o desbloquear antes del commit. |
| **Usuario PC — SQLite** | Proyección mínima offline, credencial protegida, sesión local, selecciones históricas, auditoría y hechos pendientes. | Convertirse en autoridad nacional o borrar silenciosamente conflictos. |
| **Dinamizador — renderer** | Presentar solicitudes, aprobar/rechazar cuando corresponda, observar estado y mantener el flujo manual. | Acceder directamente a SQLite, manipular PIN/verifier o implementar reglas de red. |
| **Dinamizador — proceso principal** | Gateway LAN, política de centro, autorización automática/humana, reserva y activación de sesión, unicidad de Infoplaza, distribución de proyecciones y conciliación. | Crear una tabla/modelo paralelo de “sesiones de estación” ni exponer secretos al renderer. |
| **Dinamizador — SQLite** | Registro operativo de sesiones del centro, solicitudes, estaciones, catálogos/proyecciones, inbox/outbox, auditoría y conflictos. | Sobrescribir hechos autónomos mediante last-write-wins. |
| **Plataforma central futura** | Autoridad nacional de usuario/catálogos, consolidación, credenciales/provisionamiento, correo de recuperación y conflictos entre centros. | Ser requisito de conectividad para la operación LAN básica ni imponer ahora UI, roles o reportes no definidos. |

### Autoridad por dato

- **Usuarios y catálogos nacionales:** central; Dinamizador conserva la última proyección aceptada del centro; Usuario PC conserva una proyección mínima de solo lectura.
- **Configuración operativa de Infoplaza/estación:** Dinamizador, con versión y vigencia; Usuario PC aplica la última política válida.
- **Estado observado de la estación:** originado por Usuario PC y proyectado por Dinamizador.
- **Sesiones:** un único agregado identificado por `session_id`. Usuario PC puede originar un hecho legítimo automático/autónomo; Dinamizador lo autoriza o concilia como autoridad operativa; central consolida y resuelve conflictos entre centros.
- **Auditoría:** cada componente es autor de sus propios eventos inmutables; la consolidación no cambia el evento original.

## 4. Mapa KEEP / EXTEND / REFACTOR / ADD

### KEEP

- FSM y mecanismo físico de `lockCore`/`unlockCore`, hooks, watchdog, mantenimiento y modo de ventana: el flujo normal los usa detrás del coordinador de sesión; RECOVERY/WATCHDOG y MAINTENANCE/ADMIN conservan sus rutas privilegiadas explícitas y separadas.
- Reconnector, descubrimiento mDNS y capacidad de envío/recepción WebSocket como base de transporte.
- `PanelShell`, `UserTabs` y superficie flotante del widget.
- Configuración JSON y logs técnicos rotativos existentes; no se moverán a SQLite por conveniencia.
- SQLite, migraciones, repositories e IPC estricto del Dinamizador.
- Entidades existentes `usuarios`, `pcs`, `sesiones`, `actividades` y `sync_queue` como base evolutiva.
- Flujo manual de asignación del Dinamizador como contingencia transitoria.

### EXTEND

- `connection.Client` con sobres v2, handshake, ACK y envío de outbox, conservando frames legados.
- Configuración de Usuario PC con `station_id` estable y referencias de política, sin usar `client_id`, MAC o nombre como identidad técnica.
- Sesión del Dinamizador con modalidad, origen, correlación, estado de activación, conciliación y selecciones.
- Catálogos de actividades para estado activo/cancelado, tipo semántico, fecha e Infoplaza; motivos como catálogo versionado nuevo.
- Repositorios/IPC/store de PCs para solicitudes originadas por estación y estado correlacionado.
- Widget para recibir datos reales y minimizados de la sesión.

### REFACTOR

- `protocol.go` y `dispatcher.go`: separar codec, router legado y router v2; un `UNLOCK` deja de equivaler a crear sesión en modo v2.
- Inicialización de `App`: recuperar almacenamiento y sesión antes de escoger el estado; sin sesión válida, el nuevo modo inicia bloqueado.
- Temporizador: pasar de contador en memoria a proyección calculada desde la sesión durable y checkpoints de transición.
- `sesionesRepository.iniciar`: separar servicio de aplicación, repositorio y transacción; inicio, extensión y transferencia dejan de compartir un método ambiguo.
- Escritura del núcleo de sesión + marca durable de sincronización en Dinamizador: convertirla en una transacción atómica; materializar el hecho/outbox completo de forma idempotente sin volverlo barrera de activación.
- Reglas de sync: restringir resolución genérica por timestamp a datos maestros permitidos; sesiones/auditoría usan hechos idempotentes y conflicto explícito.
- Tipos de sesión compartidos para representar el mismo agregado en flujo manual, automático, autorizado, autónomo y transferido.

### ADD

- Login y selección de motivos/opciones en Usuario PC.
- SQLite embebido, migraciones y repositorios de dominio en Usuario PC.
- Servicios de autenticación, política, sesión, auditoría, outbox/inbox, recuperación y conciliación.
- Gateway LAN y servicio de autorización/conciliación en el proceso principal de Dinamizador.
- Repositorios de solicitudes, auditoría operativa, motivos, selecciones e inbox/facts.
- Contratos versionados de capacidades, solicitudes, respuestas y eventos.

### REPLACE

No se prevé reemplazo. La documentación histórica “sin base de datos” se actualiza porque su supuesto de volumen y ausencia de relaciones ya no aplica, pero se conserva la persistencia file-based donde sigue siendo adecuada.

## 5. Modelo de datos conceptual

Todos los identificadores técnicos serán UUID/IDs internos estables. Los nombres siguientes son conceptos y contratos; el diseño de migración posterior podrá adaptar nombres físicos sin alterar semántica.

### 5.1 Identidad y estación

- **UserProjection:** para ADA Nova Plus, `user_id` es la identidad nacional global estable. `center_scope` es obligatorio para alcance operativo, elegibilidad y proyección local, pero no es parte de la identidad. La proyección incluye `user_id`, `center_scope`, estado activo/bloqueado/deshabilitado, campos públicos mínimos, señal sensible nullable `tiene_discapacidad`, versión de origen, `synced_at`, vigencia de proyección y señal `has_email`. `TRUE` significa discapacidad declarada, `FALSE` significa no declarada y `NULL` significa dato no capturado/legado; `NULL` no se interpreta como `FALSE`. No necesita correo, dirección, nacimiento, documento completo ni detalle de discapacidad para la UI pública.
- **Consecuencia de esquema PR-04A:** `user_projections` usa `PRIMARY KEY(user_id)`. `permanent_pin_credentials.user_id` usa `FOREIGN KEY(user_id) REFERENCES user_projections(user_id)` porque la credencial PIN permanente pertenece al usuario global. `UNIQUE(center_scope, document_lookup_token)` se conserva como unicidad de búsqueda operativa, sin hacer de `center_scope` un componente de identidad.
- **DocumentLookup:** token de búsqueda derivado con clave local protegida a partir del documento normalizado y tipo documental. Permite búsqueda offline sin usar el documento como PK ni conservarlo en auditoría. El documento ingresado vive solo durante el intento y, en una solicitud asistida, se transmite porque la especificación lo exige.
- **Station:** `station_id`, `center_id`, nombre visible, estado, capacidades de protocolo, versión de política y última comunicación. MAC, hostname y `client_id` son atributos diagnósticos, no identidad.

### 5.2 Credenciales y autorizaciones de PIN

Son agregados y contratos separados, con namespaces, repositorios e interfaces incompatibles por tipo; ningún adaptador puede convertir uno en otro:

- **PermanentPinCredential:** `credential_id`, `user_id`, verificador/hash no reversible, salt/parámetros/versión de algoritmo, versión, estado y timestamps. Es la única credencial usada por la autenticación ordinaria. El backend deriva el verificador con función resistente a fuerza bruta y salt único; la proyección local se protege además con DPAPI o abstracción equivalente. Nunca se conserva el PIN personal reversible.
- **TemporaryEmailPinCredential:** credencial independiente de exactamente cuatro dígitos y propósito `EMAIL_RECOVERY`, con `temporary_credential_id`, `user_id`, representación protegida para validación, `issued_at`, `expires_at`, estado y versión de emisión. La plataforma central futura genera y entrega el PIN temporal al correo registrado; Usuario PC y Dinamizador solo consumen el contrato. Tiene vigencia limitada, una sola finalización satisfactoria y la nueva emisión invalida toda temporal activa anterior del mismo usuario/propósito. No sustituye al verificador permanente ni persiste en texto plano.
- **ResetAuthorizationGrant:** autorización opaca para reemplazar la credencial permanente; no es PIN ni credencial de login. Contiene o referencia `grant_id`, modo, `user_id`, `request_id`, `center_id`, `station_id` cuando aplique, emisión/expiración, nonce o versión anti-replay y estado. Es de un uso y queda inválida por consumo satisfactorio, rechazo, expiración, cancelación o reemplazo. Su valor/secreto no se expone al renderer ni se persiste en auditoría.
- **AttemptControl:** contadores/ventanas/bloqueos asociados a IDs internos y estación. Máximo, ventana y duración provienen de política, sin constantes de negocio.

El protocolo ordinario y el IPC nunca contienen PIN actual/nuevo/temporal, verificador/hash, grant ni secreto. Los paquetes protegidos entre el backend de Usuario PC y la autoridad de credencial son opacos para Dinamizador. La única exposición permitida del PIN temporal es la entrega del servicio central al correo registrado y su captura transitoria por el usuario en la superficie de recuperación.

### 5.3 Catálogos

- **MotiveCatalogItem:** `motive_id`, código semántico estable, etiqueta, activo, orden, versión y vigencia. El código semántico, no el texto traducido, dispara la consulta de capacitación/actividad.
- **ScheduledOption:** `option_id` estable, tipo `training|activity`, `center_id`, fecha/ventana, estado activo/cancelado, nombre mínimo y versión. Se adapta a las actividades existentes del Dinamizador en lugar de duplicarlas.
- **DisabilityCatalogItem:** `disability_type_id`, código estable, etiqueta, activo, orden, versión y vigencia. Es configurable por catálogo; no se codifica en UI/backend y no se guarda como texto libre en `usuarios`.
- **UserDisabilitySelection:** relación normalizada entre `user_id` y `disability_type_id`. Si `tiene_discapacidad` es verdadero, debe existir al menos una relación activa conforme a la cardinalidad que se apruebe; si es falso, no se conservan asociaciones activas; si es `NULL`, representa dato desconocido y no se inventan asociaciones. Sus valores son dato personal sensible y no se proyectan en pantallas públicas.
- **CatalogSnapshot:** versión, autoridad, fecha de recepción y validez. Usuario PC usa el último snapshot válido completo; un snapshot parcial no reemplaza silenciosamente el último válido.
- **SessionMotiveSelection / SessionOptionSelection:** enlaces con IDs originales y snapshot de código/etiqueta/tipo/fecha necesarios para que el historial no cambie cuando cambie el catálogo. Se permite múltiples motivos y opciones; cero opciones programadas se registra explícitamente como resultado de consulta, no como motivo faltante.

### 5.4 Sesión única

- **Session:** `session_id`, `center_id`, `user_id`, estación actual, inicio, duración concedida, fin esperado/checkpoint, estado, modalidad `automatic|authorized|autonomous`, origen `station|dinamizador_manual|transfer`, `request_id`, versión de política, estado de sincronización/conciliación, timestamps y versión.
- **SessionStationHistory:** conserva estaciones origen/destino y tiempos de asignación usando el mismo `session_id`; una transferencia no crea otra sesión.
- **PR-05B1b queda SPLIT / SUPERSEDED y no tiene superficie ejecutable propia.** PR-05B1b1 fija la finalización durable pendiente: el ciclo de activación local de `Session.status` es `pending_unlock` → `active` o `pending_unlock` → `unlock_failed`, y `PersistSessionActivation` confirma siempre el agregado nuevo exactamente como `pending_unlock`. PR-05B1b2 coordina después la barrera física; el claim y la intención existen duraderamente, pero la estación sigue bloqueada y nunca se persiste `active` antes de confirmar el desbloqueo físico. `pending_activation` conserva, cuando corresponda, su significado de reserva remota anterior a esta activación local; `finished`, `interrupted`, `pending_reconciliation` y `conflict` pertenecen al ciclo posterior. La API pública puede proyectar los estados existentes mientras dura la migración.
- **SessionBlockingClaim (`session_blocking_claims`):** cada claim pertenece a un `session_id` y reserva de forma única `station_id` y `(center_id,user_id)`. Se inserta atómicamente con `Session`, motivos y `SyncIntent`; es independiente de `Session.status` y no tiene TTL, expiración, renovación ni liberación por reloj.
- PR-05A2 cubre solo adquisición, rechazo de duplicados, rollback, integridad referencial y prevención de huérfanos del claim; PR-05A1 aporta únicamente su esquema aditivo. Su liberación se difiere al ciclo de vida posterior y no se define en este cambio.
- `session_id` no cambia en reintentos, reinicios, activación tardía ni transferencia. El legado `sesion_original_id` se conserva para datos históricos, pero no define el nuevo agregado.

### 5.5 Auditoría y hechos pendientes

- **AuditEvent de negocio/sesión:** `event_id`, tipo, `occurred_at`, resultado, IDs internos aplicables, modalidad, origen, `correlation_id` y detalles estructurados permitidos. Es append-only y no sustituye el `SecurityAuditSink` obligatorio del vínculo PR-08.
- **SyncIntent:** conserva exactamente `sync_intent_id`, `aggregate_type`, `aggregate_id`, `version` y `status`; PR-05A1 define su esquema y PR-05A2 admite únicamente `aggregate_type = session`, sin otra taxonomía. `sync_intent_id` es su identidad y clave única; no se exige unicidad de `(aggregate_type, aggregate_id, version)`.
- `SyncIntent.version` es exclusivamente la revisión durable local del propio `SyncIntent`: `INTEGER` positivo, inicial `1`, propiedad de su persistencia/ciclo de vida local. PR-05A2 solo crea versión `1`, sin updates ni incrementos. No representa revisión de sesión, origen, política, payload, protocolo o remoto; tampoco es contador de reintentos, precedencia ni entrada de selección. Una futura mutación del mismo `SyncIntent` deberá incrementar atómicamente su revisión, pero su semántica permanece indefinida y fuera de alcance.
- **LocalFact:** `fact_id`, tipo/version de esquema, agregado/ID, origen, secuencia o versión, modalidad, payload minimizado, fecha y procedencia.
- **OutboxItem:** referencia al hecho, estado `pending|sent|acknowledged|conflict|rejected`, intentos, próximo intento y último código de error.
- **InboxReceipt:** `(origin_id,message_or_fact_id)` único para deduplicar reentregas.
- **ConciliationCase:** IDs de hechos involucrados, autoridad requerida, estado y razón; nunca se resuelve mediante overwrite silencioso.

La transacción previa al desbloqueo contiene el núcleo de sesión —ID, usuario, estación, modalidad, motivos, tiempos iniciales y estado— más `SessionBlockingClaim` y `SyncIntent`. Las selecciones secundarias, `LocalFact`, `OutboxItem` y `AuditEvent` pueden acompañarla cuando el costo sea acotado, pero sus errores aislables se capturan sin abortar el commit nuclear; si se omiten o fallan, un worker recuperable los materializa idempotentemente desde el núcleo y la intención. En Dinamizador se aplica la misma separación entre núcleo/marca y artefactos secundarios.

## 6. Persistencia local de Usuario PC

### Decisión

Se añadirá una base **SQLite embebida** bajo `%PROGRAMDATA%/Infoplaza Agente/data/`, separada de `config.json` y de los logs técnicos. Se accederá mediante `database/sql` y un driver compatible con el empaquetado Windows; se prefiere un driver Go puro para no introducir una dependencia de CGO en el instalador. La selección final del driver debe validarse con un spike de build antes de implementación, sin cambiar el contrato de repositorios.

SQLite es la opción mínima que satisface simultáneamente relaciones, filtros por fecha/Infoplaza, índices, constraints, migraciones, una marca de sincronización recuperable y el commit atómico del núcleo antes de desbloquear. Archivos JSON independientes requerirían implementar locking, journaling, atomicidad multiarchivo y recuperación, generando más complejidad y riesgo.

### Operación

- Una migración versionada y transaccional crea/actualiza el esquema; no se depende de la versión implícita de `config.json`.
- Se habilitan foreign keys y WAL. La configuración de sincronización debe priorizar confirmación durable para la transacción crítica de sesión; no se considerará commit una escritura aún no confirmada por SQLite.
- Un servicio de persistencia serializa las escrituras críticas y expone interfaces (`UserRepository`, `CredentialRepository`, `SessionRepository`, `CatalogRepository`, `AuditRepository`, `OutboxRepository`) para pruebas.
- La clave local usada para proteger campos/verifiers se guarda mediante DPAPI, reutilizando la abstracción existente de protección, no en el mismo archivo SQLite en claro.
- Al arrancar: abrir/migrar DB, recuperar primero sesión y pendientes, validar identidad/configuración según la clase de política, resolver el temporizador y solo entonces seleccionar FSM. Una sesión activa recuperable continúa con su último estado/configuración válidos; un error de DB o núcleo ambiguo que impida demostrar esa sesión mantiene la estación bloqueada y registra la razón, sin inventar otra sesión.
- Una sesión activa recuperable restaura widget/temporizador desde timestamps y checkpoints durables. La ausencia de configuración nueva no interrumpe esa continuidad; solo una incoherencia del núcleo que no permita reconstruirla requiere contención y conciliación.

## 7. Integración SQLite de Dinamizador

El modelo físico existente `sesiones` se migra de forma aditiva para incluir modalidad, origen, correlación/idempotencia, estado de activación, conciliación y versión de política. Tablas de enlaces conservan motivos/opciones y trazabilidad de estaciones. Solicitudes de autorización y recuperación son agregados auxiliares, no sesiones paralelas.

`sesionesRepository` se divide conceptualmente en:

1. **SessionRepository:** lecturas/escrituras puras dentro de una transacción provista.
2. **SessionApplicationService:** aplica unicidad, reserva, activación, flujo manual, transferencia y conciliación.
3. **LAN handlers:** validan sobre v2 y llaman al mismo servicio que IPC.
4. **IPC handlers:** llaman al mismo servicio para el flujo manual.

Así, una asignación manual y una solicitud desde estación pasan por las mismas restricciones e inserciones. El renderer nunca recibe verifier, grants secretos ni paquetes de credencial.

La cola existente se conserva, pero los hechos de sesión usan IDs estables y estados de aceptación/conflicto. `resolveConflict()` no se aplicará a sesiones, auditoría ni transferencias. El intervalo de persistencia de cronómetro y la duración predeterminada se obtienen de settings/policy; los valores actuales no se elevan a reglas normativas.

## 8. Contrato LAN v2 y compatibilidad

### 8.1 Decisión de alcance: PR-07 no establece confianza

**PR-07 entrega únicamente el sobre Agent v2, su codec de encode/decode, el discriminador sin downgrade, la separación segura del parser legado y deduplicación de transporte en memoria.** El WebSocket actual continúa sin autenticación; por ello, incluso un sobre v2 estructuralmente válido es entrada no confiable y su routing permanece inerte para todo efecto privilegiado.

| PR-07 — vinculante ahora | PR-08+ — arquitectura futura preservada, no implementable en PR-07 |
| --- | --- |
| Decodificar, validar y clasificar sobres v2; codificarlos para pruebas y emisores futuros. | Handshake/capacidades y vínculo autenticado e íntegro entre estación y Dinamizador. |
| Elegir de forma irreversible parser v2 o parser legado por presencia de marcadores. | Autoridad de `link_id`, binding de `station_id`/`center_id` y privilegios del peer autenticado. |
| Deduplicar frames v2 por `message_id` dentro de una conexión. | Obligatoriedad contextual y monotonicidad de `connection_epoch`+`sequence`, frescura y anti-replay. |
| Entregar v2 válido solo a un callback inerte/unsupported, sin efectos de producto. | Tipos productivos, schemas de negocio, idempotencia por operación, ACKs productivos y activación/conciliación. |
| Mantener sin cambios el decoder y comportamiento legado existentes. | Autorización, sesión, Coordinator, persistencia, temporizador y sender del Dinamizador. |

PR-07 excluye autenticación, vínculo confiable, privilegios, `ACTIVATE_SESSION` productivo, `SessionActivation`, Coordinator, persistencia, efectos de temporizador, resolución de recuperación y sender del Dinamizador. No introduce DTO de activación, vocabulario de resultados como `approved`, `conflict` o `physical_failure`, ACK automático ni mensaje productivo alguno. Tampoco autentica al emisor, concede confianza por parsear un frame ni habilita el login v2; PR-08 a PR-11 conservan la propiedad de esas capacidades futuras.

### 8.2 Sobre Agent v2 cerrado para PR-07

El frame v2 MUST ser un objeto JSON. Los campos conocidos tienen este contrato estructural:

| Campo | Presencia en PR-07 | Tipo y validación | Semántica PR-07 |
| --- | --- | --- | --- |
| `protocol_version` | Obligatorio | Entero JSON exactamente `2`. Ausente o de tipo incorrecto en un candidato v2 es `malformed_v2`; cualquier otro entero es `unsupported_protocol_version`. | Selecciona el codec Agent v2; nunca habilita privilegios. |
| `message_id` | Obligatorio | String no vacío ni compuesto solo por whitespace. | Identidad única del frame de transporte y única clave de deduplicación PR-07; no es `session_id` ni identidad idempotente de negocio. |
| `kind` | Obligatorio | String exactamente `request`, `response`, `event`, `command` o `ack`; cualquier otro valor se rechaza estructuralmente como `malformed_v2`. | Clasificación estructural sin interpretación privilegiada. |
| `type` | Obligatorio | String no vacío ni compuesto solo por whitespace. | PR-07 no define un registro completo. Un tipo desconocido/no registrado puede ser v2 válido, pero su routing es inerte/unsupported y no produce efectos. |
| `correlation_id` | Obligatorio para `response` y `ack`; opcional para `request`, `command` y `event`, incluidos eventos causados | Si está presente, string no vacío ni compuesto solo por whitespace. | Correlación de mensajes; no tiene semántica de sesión. |
| `idempotency_key` | Opcional | Si está presente, string no vacío ni compuesto solo por whitespace. | Solo decode/validate. No participa en deduplicación PR-07; schemas futuros decidirán cuándo es obligatorio por operación. |
| `source_id`, `station_id`, `center_id` | Opcionales en el sobre base PR-07 | Si están presentes, strings no vacíos ni compuestos solo por whitespace. | Datos declarados no confiables. No se transforman; `station_id` nunca se sustituye por `ClientID`, MAC o hostname. |
| `sent_at` | Obligatorio | String RFC3339; se acepta precisión RFC3339Nano y se exige `Z` o un offset de zona válido. | Solo decode/validate. PR-07 no aplica frescura, skew ni anti-replay. |
| `connection_epoch` | Opcional | Entero JSON sin signo representable como `uint64`; se rechazan negativos, fracciones, overflow y tipos no numéricos. | Solo decode/validate. |
| `sequence` | Opcional | Entero JSON sin signo representable como `uint64`; se rechazan negativos, fracciones, overflow y tipos no numéricos. | Solo decode/validate. Es la única alternativa de orden del sobre; no existe campo `nonce` alternativo. |
| `link_id` | Opcional | String no vacío ni compuesto solo por whitespace. | Solo decode. Es no confiable y no autoriza nada en PR-07. |
| `payload_schema_version` | Obligatorio | Entero JSON positivo. Solo `1` está soportado; otro positivo es `unsupported_payload_schema`, y tipo inválido o valor no positivo es `malformed_v2`. | Versiona únicamente la forma futura del payload. |
| `payload` | Obligatorio | JSON sintácticamente válido, conservado opaco como `json.RawMessage` o equivalente. | PR-07 no interpreta schemas de sesión o negocio. |

Todos los identificadores del sobre —`message_id`, `correlation_id`, `idempotency_key`, `source_id`, `station_id`, `center_id` y `link_id`— son strings opacos: no tienen restricción UUID, no se normalizan, recortan, cambian de caso ni sustituyen. La comprobación de whitespace sirve solo para aceptar/rechazar; el valor aceptado se conserva byte-semánticamente como string.

Los campos JSON desconocidos se toleran para compatibilidad futura, pero nunca alteran routing, clasificación o privilegios y no relajan la validación de campos conocidos. La política sobre claves JSON duplicadas queda explícitamente diferida: `encoding/json` no ofrece rechazo estricto automático y no existe requisito aprobado para incorporarlo en PR-07.

### 8.3 Discriminador sin downgrade

El selector inspecciona **presencia de claves**, no valores truthy ni éxito parcial de decode. Los marcadores exclusivos de candidatura v2 son:

```text
protocol_version, message_id, kind, correlation_id, idempotency_key,
source_id, station_id, center_id, link_id, connection_epoch, sequence,
sent_at, payload_schema_version
```

`type` y `payload` no son marcadores porque pertenecen también al frame legado. El flujo de clasificación es vinculante:

1. Un frame que no sea JSON sintácticamente válido o no sea un objeto se rechaza sin efecto. Si es un objeto JSON, se inspeccionan sus claves preservando los valores crudos para el codec elegido.
2. La presencia de `protocol_version` **o de cualquier marcador exclusivo** hace al frame candidato v2.
3. Un candidato v2 pasa exclusivamente al decoder v2. Nunca se intenta el decoder legado después, aunque falte `protocol_version`, tenga tipo incorrecto, declare una versión no soportada, falle `payload_schema_version` o sea estructuralmente inválido.
4. En el camino v2, `protocol_version` ausente/tipo inválido es `malformed_v2`; otro entero es `unsupported_protocol_version`; schema positivo distinto de `1` es `unsupported_payload_schema`; cualquier otra violación estructural conocida es `malformed_v2`.
5. Solo un objeto sin ningún marcador v2 puede pasar al decoder legado existente.

Así, por ejemplo, `{"type":"UNLOCK","message_id":"x"}` es candidato v2 malformado por ausencia de `protocol_version`; jamás se degrada a `UNLOCK` legado.

### 8.4 Camino legado preservado

El camino sin marcadores conserva el frame `{ "type": "...", "payload": ... }`, el decoder actual y el comportamiento vigente en la base de PR-07 para `UNLOCK`, `LOCK`, `SET_TIMER`, `MESSAGE`, `SHUTDOWN` y `CANCEL_MAINTENANCE`. Esto incluye el rechazo actual de comandos desconocidos. PR-07 no amplía, corrige ni reinterpreta payloads legados y no revierte las barreras que PR-05B2 ya haya establecido.

La compatibilidad es una selección de parser, no una heurística de recuperación: un error v2 nunca prueba suerte con `ParseMessage`, y un `type` v2 desconocido nunca se convierte en comando legado.

### 8.5 Flujo de datos y frontera inerte v2

```text
WebSocket frame
  → inspección JSON + discriminador
      → sin marcadores v2: decoder legado → flujo legado vigente
      → candidato v2: codec v2 → validación → dedupe por message_id
                                         → callback v2 inerte/unsupported
                                         ↛ Dispatcher legado
                                         ↛ FSM/EventUnlock/EventLock privilegiado
                                         ↛ SessionActivationCoordinator/SQLite/temporizador
```

Un frame v2, incluso con `kind=command` y `type=UNLOCK`, `LOCK`, `SET_TIMER`, `SHUTDOWN` o `CANCEL_MAINTENANCE`, MUST NOT reutilizar `Dispatcher` ni alcanzar `EventUnlock`, el comportamiento privilegiado de `EventLock`, temporizador, apagado, mantenimiento, Coordinator, persistencia o `SessionActivation`. `MESSAGE` tampoco adquiere un efecto de UI por coincidir con un nombre legado. El callback v2 de PR-07 puede observar clasificación/metadatos o devolver `unsupported`, pero no ejecutar producto.

Frames v2 malformados o con versión/schema no soportados no causan panic, fallback legado, efecto privilegiado, terminación del proceso ni reconexión por sí solos. El read loop registra el resultado mínimo y continúa con el siguiente frame. Los errores de red, cierre o socket conservan el comportamiento vigente de desconexión/reconexión.

### 8.6 Deduplicación de transporte PR-07

- Clave única: `message_id`; `idempotency_key` no participa.
- Scope: una conexión WebSocket activa.
- Almacenamiento: memoria, capacidad fija `256`, orden FIFO oldest-first.
- Inserción: solo después de validar por completo un sobre v2. Un duplicado no refresca su posición FIFO.
- Resultado: un `message_id` válido ya visto se clasifica `duplicate`, no se entrega por segunda vez al callback/routing, no tiene efecto privilegiado, no es error fatal de conexión y no genera ACK automático.
- Evicción: al insertar el elemento 257 se elimina el más antiguo; un ID evicto puede volver a tratarse como nuevo dentro de esa misma conexión.
- Reinicio: una conexión WebSocket nueva, incluida una reconexión, empieza con cache vacío.

No existe dedupe durable o entre reconexiones, idempotencia de sesión ni reutilización de `idempotency_key` en PR-07.

### 8.7 Codec bidireccional y categorías observables

El mismo modelo/codec v2 soporta decode y encode. Las pruebas de round-trip incluyen obligatoriamente `kind=response` y `kind=ack`, ambos con `correlation_id`; codificar un sobre que viola esa condición falla. Encode también valida payload JSON y las mismas restricciones conocidas antes de emitir bytes. PR-07 no emite ACKs de producto automáticamente.

Las categorías mínimas distinguibles son `valid_v2`, `duplicate`, `malformed_v2`, `unsupported_protocol_version`, `unsupported_payload_schema`, `unsupported_type` y el resultado legado vigente. `unsupported_type` presupone un sobre estructuralmente válido: no autoriza nada y puede ser el resultado del callback/routing inerte, no una razón para degradar a legado.

### 8.8 Logging y hardening diferido

Nunca se registra el frame v2 crudo ni el `payload` completo. Solo se permiten versión, `kind`, `type`, resultado/categoría del parser y, si la política local lo permite, `message_id`/`correlation_id`. Los campos desconocidos y valores internos del payload no se promueven a logs por conveniencia.

PR-07 no agrega un umbral arbitrario de tamaño de frame/read. Límites de transporte, protección de memoria y política de cierre por frames excesivos quedan como follow-up explícito de hardening, separado de este wire contract.

### 8.9 Impacto, pruebas y rollout de PR-07

La implementación posterior se concentra en `Soft_Usuario_PC/agente-infoplaza/internal/connection`: `protocol.go` o un archivo de codec v2 enfocado define tipos, validación, encode y discriminación; `client.go` integra clasificación, continuidad del read loop y cache FIFO por conexión. `protocol_test.go` y `client_test.go` cubren el contrato. `internal/commands/dispatcher.go` no recibe sobres v2; solo requiere regresiones que demuestren que su camino legado permanece aislado. PR-07 no modifica paquetes de sesión, Coordinator, persistencia, temporizador ni Dinamizador productivo.

La matriz mínima de pruebas incluye:

- cada campo obligatorio/condicional, tipos JSON erróneos, whitespace, enteros negativos/fraccionarios/overflow, versiones/schema no soportados y timestamps RFC3339/RFC3339Nano con zona;
- todos los marcadores de candidatura por separado, en especial candidatos sin `protocol_version`, y prueba de que ningún error v2 invoca legado;
- unknown fields tolerados y tipos v2 desconocidos válidos pero inertes;
- payload opaco válido, payload ausente o JSON inválido, sin DTO de negocio;
- round-trip de `request`, `response`, `event`, `command` y `ack`, con correlación obligatoria para `response`/`ack`;
- duplicado sin segunda entrega, capacidad 256, evicción FIFO, no refresh por duplicado y reset al reconectar;
- frames v2 llamados `UNLOCK`, `LOCK`, `SET_TIMER`, `MESSAGE`, `SHUTDOWN` o `CANCEL_MAINTENANCE` sin acceso a Dispatcher/FSM/Coordinator ni efecto observable de producto;
- error de parse que no termina el proceso, no reconecta por sí solo, no impide leer el frame siguiente y no filtra payload crudo al log;
- regresión del frame legado y de su rechazo actual de comando desconocido.

El rollout de PR-07 es seguro por construcción: puede desplegarse antes de PR-08 porque todo routing v2 es inerte y su dedupe es efímero. Rollback restaura el binario anterior sin migración ni estado durable que revertir; la cache se pierde deliberadamente al cerrar la conexión/proceso. La habilitación de handshake, vínculo confiable o cualquier efecto v2 queda prohibida hasta PR-08+.

### 8.10 PR-08 — arquitectura de vínculo seguro aprobada

Los nueve bloqueadores técnicos originales están **CLOSED** en [`pr-08-blocking-requirements-closure.md`](./pr-08-blocking-requirements-closure.md). El contrato normativo cross-component está en [`specs/dinamizador-usuario-pc-secure-link/spec.md`](./specs/dinamizador-usuario-pc-secure-link/spec.md). El cierre habilita planificación, no `apply` ni worktrees. PR-07 permanece COMPLETE y aporta solo el sobre/routing v2 inerte; PR-08 construye confianza encima sin habilitar acciones de producto.

#### Transporte y PKI

- El tráfico LAN privilegiado confiable usa exclusivamente WSS sobre TLS 1.3 con mTLS: Usuario PC cliente, Dinamizador servidor. No se permite WS en claro, contraseña global/compartida ni handshake criptográfico custom.
- Una CA privada controlada por Infoplazas emite una identidad/certificado único por instalación. La clave firmante de CA nunca reside en endpoints.
- La clave de endpoint se genera en dispositivo y se enrola por CSR mediante emisión online autorizada o paquete offline firmado (vinculado a CSR/nonce exacto y estado monotónico). La raíz pública se fija localmente y runtime opera sin Internet con credencial no expirada y estado local conocido.
- Claves privadas usan storage Windows protegido y ACL restrictiva. Un spike seleccionará Certificate Store/CNG o DPAPI protegido; DPAPI solo pasa si demuestra que no hay texto plano, exportación o copia y que la clave nunca sale de la instalación. El diseño no selecciona adaptador antes del spike.

#### Identidad y ciclo de vida

La identidad de estación y Dinamizador exige simultáneamente certificado válido, registro de instalación autorizado y binding al centro. `station_id`, `center_id`, `source_id` y `link_id` siguen siendo claims hasta cross-check; operador humano es una identidad posterior y separada. IP, MAC, hostname y mDNS no confieren confianza.

Certificados tienen vigencia finita. Renovación/reemisión de certificado se distingue de rotación de clave/nuevo enrolamiento. Reinstalación/reemplazo genera clave nueva y deshabilita/revoca la anterior. Overlap temporal solo representa una instalación bajo cutover explícito; la credencial antigua se niega tras cutover o expiración. Ambos endpoints conservan peers autorizados/deshabilitados y Dinamizador mantiene deny local de estaciones. Revocación central sincroniza cuando existe conectividad; revocación global inmediata es imposible totalmente offline y queda como limitación explícita. Estado local conocido inválido/revocado/deshabilitado falla cerrado. PR-08 no fija cadencia de rotación.

#### FSM y data flow

```text
mDNS/configuración local (descubrimiento, no identidad)
  → TCP: CONNECTED_UNAUTHENTICATED
  → TLS 1.3 mTLS + WebSocket upgrade: AUTHENTICATING
  → WSS autenticado + registro/centro/capacidades pendientes:
      AUTHENTICATED_NEGOTIATING
  → link.hello
  → validación de certificado + registro + centro + versión + capacidades
      ├─ link.reject cuando exista canal/correlación segura → REJECTED/CLOSING
      └─ link.accept verificado + epoch nuevo → ACTIVE
  → guard post-ACTIVE: epoch + secuencia + link_id opcional + allowlist
      → elegibilidad técnica únicamente
      ↛ handler o acción de producto en PR-08
```

Cada conexión/reconnect comienza sin auth, época, secuencia, capacidades o privilegios heredados; reemplazar peer invalida el binding. TLS 1.3 aporta prueba mutua/transcript. Early data/0-RTT se deshabilita. Resumption solo se admite si cada conexión revalida vigencia, identidad, autorización, centro y deny local; de otro modo se deshabilita.

#### Contrato de negociación

Se reutiliza el sobre PR-07 sin duplicar campos:

| Tipo | Forma exacta |
| --- | --- |
| `link.hello` | Estación→Dinamizador; `kind=request`; protocolo `2`, schema `1`; `message_id`, `sent_at`, claims `source_id/station_id/center_id`; sin epoch, sequence, `link_id` ni `idempotency_key`; payload exacto `{build_version: string no vacío, capabilities: string[] único/no vacío}` con `secure_link_v1`. |
| `link.accept` | Dinamizador→estación; `kind=response`; correlación exacta al hello; claims vinculados; epoch aleatorio criptográfico no cero `uint64`; `link_id` no secreto opcional; sin sequence/idempotency; payload exacto `{negotiated_capabilities: string[]}` igual a la intersección conocida allowlisted e incluyendo `secure_link_v1`. |
| `link.reject` | Solo después de mTLS y hello estructuralmente utilizable/correlacionable; `kind=response`, correlación exacta; sin epoch/sequence/ACTIVE; payload con una categoría aprobada. |

Solo accept completamente verificado activa. Versión exacta `2/1`; no hay downgrade privilegiado a v1. La capability registry es explícita/allowlisted, `secure_link_v1` es obligatoria y PR-08 anuncia cero capacidades de negocio.

#### Rechazo, replay y tiempo

Fallo TLS se audita localmente —`AUTH_REQUIRED`, `CERT_INVALID`, `CERT_EXPIRED`, `CERT_REVOKED`, con not-yet-valid/untrusted-CA/TLS-failure como subrazones cuando exista detalle— y no produce JSON; TLS alert no es `link.reject`. Después de mTLS, las categorías wire elegibles según contexto son `IDENTITY_MISMATCH`, `CENTER_MISMATCH`, `PROTOCOL_UNSUPPORTED`, `SCHEMA_UNSUPPORTED`, `CAPABILITY_REQUIRED`, `HANDSHAKE_MALFORMED`, `HANDSHAKE_TIMEOUT`, `REPLAY`, `SEQUENCE_INVALID`, `PEER_REPLACED` y `OUT_OF_SCOPE`. Timeout requiere correlación segura; incompatibilidad puede auditarse y cerrar sin response. `REJECTED` no implica frame enviado.

Dinamizador genera por conexión exitosa un `connection_epoch` criptográficamente aleatorio, no cero, `uint64`, no persistido/reutilizado. Handshake carece de secuencia. Post-ACTIVE, cada dirección usa secuencia `uint64` desde `1` y acepta solo `+1`; duplicado/menor es replay, gap mayor audita+cierra+reautentica, reconnect reinicia bajo época nueva y se cierra antes de overflow. `link_id`, si existe, debe coincidir. El FIFO PR-07 por `message_id` permanece dedupe de transporte separado.

`sent_at` es observacional. X.509 usa reloj OS local sin Internet/NTP; reloj implausible impide nuevo ACTIVE y la conexión cierra no después de `notAfter`. Rollback absoluto de reloj no puede impedirse offline sin tiempo confiable y se registra como limitación.

#### Auditoría y guard privilegiado

`SecurityAuditSink` es local/durable y allowlisted: `event_id`, `event_category`, `timestamp`, `result`, IDs confiables cuando se conocen, versiones, resumen de capacidades, `link_id` no secreto y rechazo. Excluye secretos, credenciales, claves privadas, material público innecesario, PIN/verificadores, frames/payloads crudos, payload de negocio opaco, PII sensible y discapacidad. Fallo del sink nunca concede; auditoría obligatoria no disponible mantiene elegibilidad privilegiada falsa, aunque el link pueda seguir autenticado para operación no privilegiada. Retención/acceso/exportación siguen como política de negocio.

Auth es necesaria, no suficiente. Un futuro handler exige vínculo actual, identidad/centro, versión/capability, epoch, secuencia fresca, operación allowlisted y autorización de workflow. Parser legado puede permanecer; mutación privilegiada legada no autenticada nunca ejecuta y no existe v2-failure→legacy downgrade.

#### Alcance, archivos y rollout PR-08

PR-08 habilita cero handlers, migraciones, `SessionActivation`, Coordinator, persistencia o `ACTIVATE_SESSION`. Las superficies futuras se mantienen enfocadas:

- Dinamizador A1/A2: contratos/FSM/capability/version y guard epoch/sequence/replay, preferentemente en módulos nuevos de main/shared-types con tests sin composición de gateway.
- Dinamizador B1: gateway WSS/TLS, adaptadores de certificado/registro/centro/lifecycle/clock/resumption en main process.
- Dinamizador B2: adapter `SecurityAuditSink` durable y composición Electron; renderer/IPC no reciben claves.
- Usuario PC U1: storage protegido, carga de credencial, cliente WSS/mTLS y peer binding en `internal/connection`/módulo security enfocado.
- Usuario PC U2: negociación/FSM, epoch-sequence/reconnect y audit adapter; el Dispatcher y la sesión permanecen fuera.

Los umbrellas PR-08A, PR-08B y PR-08U son no ejecutables/superseded por A1/A2, B1/B2 y U1/U2. A suma 400–510, B 380–510, U 480–630; Dinamizador 780–1,020 y cross-repo 1,260–1,650. Cada slice target máximo es ≤400 y no existe excepción 401–450. El primer worktree potencial es A1 después de preflight nuevo; B1 requiere A integrado/verificado; U1 requiere freeze A1/A2 y spike de storage aceptado, y puede avanzar en paralelo con B. Conformidad A+B+U precede PR-11; PR-09/10 siguen inertes/capa de sesión.

## 9. Flujos de inicio

### 9.1 Preparación común

1. UI normaliza formato y exige documento, PIN de exactamente cuatro dígitos y luego uno o más motivos.
2. Backend busca `user_id` mediante el token local, valida estado, credencial y política de intentos sin revelar si falló documento o PIN.
3. El PIN se elimina del estado de UI y memoria reutilizable tan pronto como sea práctico; no se registra ni se envía.
4. Se consultan opciones activas por código semántico, fecha e Infoplaza. La ausencia se informa y permite continuar.
5. El coordinador comprueba sesión local de estación/usuario y consulta o usa la proyección operativa disponible. Un duplicado conocido recupera/continúa la sesión; nunca crea otra.
6. Se crea un `request_id` estable para el intento lógico. Reintentos conservan ese ID.

### 9.2 Automática

- La política válida indica `automatic`; no hay aprobación humana individual.
- Con LAN disponible, Dinamizador realiza automáticamente la comprobación de unicidad y crea una reserva `pending_activation` con `session_id` estable.
- Sin LAN, la estación puede iniciar bajo la última política automática válida y confirmar una intención pendiente de conciliación; la modalidad sigue siendo `automatic`, no `autonomous`.
- Usuario PC confirma en una transacción el núcleo de sesión, sus motivos y `SyncIntent`, con `Session.status = pending_unlock`. Después, solo `SessionActivationCoordinator` inicia la proyección del temporizador y envía `EventUnlock`; opciones secundarias, hecho/outbox y auditoría se anexan en esa transacción solo si no penalizan la activación o se materializan idempotentemente después.
- Envía `session.activated`; reenvíos actualizan el mismo agregado.

### 9.3 Autorizada

1. Usuario PC envía `session.start.request` y permanece bloqueado.
2. Dinamizador guarda la solicitud idempotente y la presenta al operador.
3. Rechazo/cancelación termina la solicitud sin crear sesión activa.
4. Aprobación crea o reutiliza una reserva `pending_activation` y devuelve el mismo `session_id`, duración aprobada/predeterminada y versión de decisión.
5. Usuario PC verifica que la solicitud siga vigente, confirma la transacción local crítica como `pending_unlock` y recién entonces el coordinador envía `EventUnlock`.
6. `session.activated` promueve la reserva a activa. Si se pierde el ACK, el reintento devuelve el mismo resultado. Si falla la persistencia local, la estación sigue bloqueada y reporta `session.activation_failed`; la reserva no se convierte silenciosamente en otra sesión.
7. Una respuesta tardía para solicitud terminal/cancelada se archiva y audita, pero no activa nada.

### 9.4 Autónoma

Se evalúa únicamente si la política de centro es `authorized`, la función autónoma está autorizada, falla la LAN con Dinamizador después de comprobaciones/reintentos configurados y el usuario cumple elegibilidad offline. Perder solo Internet no cambia el flujo autorizado.

La estación usa última política, duración y catálogos válidos, confirma el núcleo durable con modalidad `autonomous`, `Session.status = pending_unlock` y `SyncIntent` pendiente; solo entonces el coordinador envía `EventUnlock`. El worker materializa el hecho/outbox recuperable; al regresar LAN presenta el mismo `session_id/fact_id` y Dinamizador reconoce, concilia o abre conflicto.

No se codifican heartbeat, reintentos, gracia ni umbral. Un objeto `ConnectivityPolicy` validado suministra esos valores. Al ser configuración necesaria para crear una operación autónoma nueva, su ausencia/invalidez bloquea solo ese inicio y registra la razón; no interrumpe sesiones ya activas.

### Riesgo de unicidad durante aislamiento

Una estación aislada puede impedir duplicados locales y duplicados conocidos en la última proyección, pero no puede demostrar que el mismo usuario no inicia simultáneamente en otra estación también aislada sin un coordinador compartido. Se aprueba tratar ese límite mediante prevención local/proyectada, conservación de ambos hechos y conflicto explícito, nunca overwrite. La autonomía conserva feature flag y observabilidad como control de rollout; no se inventará un lease o consenso P2P silenciosamente y este riesgo aceptado ya no bloquea tareas.

## 10. Durabilidad, temporizador e idempotencia

### Subdivisión de la barrera de desbloqueo

PR-05 conserva su división atómica: PR-05A1 (esquema) y PR-05A2 (persistencia nuclear). **PR-05B1a y PR-05B1b quedan SPLIT / SUPERSEDED**, sin superficie ejecutable propia, por la continuación aprobada en este orden estricto: **PR-05B1a1 — propagación de errores FSM `OnEnter`** → **PR-05B1a2a — observabilidad de postcondición de desbloqueo de ventana** → **PR-05B1a2b — observabilidad de hook, `unlockCore` y watchdog** → **PR-05B1b1 — transiciones de repositorio y semántica de conflicto** → **PR-05B1b2 — coordinador de activación de sesión y barrera física de desbloqueo** → **PR-05B2 — barreras legadas y cableado de función/configuración** → **PR-06 — recuperación tras reinicio y temporizador durable** → **PR-07 — sobre v2 y separación segura del parser legado**. Esta es una subdivisión de tamaño y límites de entrega; no altera las decisiones funcionales aprobadas.

**PR-05B1a1** es exclusivamente la propagación de errores FSM. `StateEnterCallback` devuelve `error`; `OnEnter` es error-capable; se preservan `from`/`to` correctos; `Send` devuelve el error de `OnEnter` y preserva `errors.Is`. El estado destino queda committed tras el error, no hay rollback automático, y las pruebas semánticas FSM cubren éxito, error, `errors.Is`, destino preservado, `from`/`to` correctos, exactamente una invocación y transición inválida que omite el callback. Su único puente de compilación es mínimo en `app_lifecycle.go`: `OnEnter` adopta la firma nueva y retorna `nil`. No propaga todavía errores físicos reales de desbloqueo. Forecast: **110–115 líneas modificadas**.

**PR-05B1a2a** es exclusivamente la observabilidad de la postcondición de ventana. `WindowManager.SetWidgetMode` elimina fullscreen y verifica `runtime.WindowIsFullscreen`; fullscreen todavía activo o estado crítico `UNKNOWN` es FAILURE, y el bookkeeping interno solo ocurre tras una verificación runtime coherente. Always-on-top, resize y center continúan best-effort. Sus pruebas de `WindowManager` cubren fullscreen falso exitoso, fullscreen verdadero como fallo, estado crítico no verificable como fallo, operaciones cosméticas best-effort y que el bookkeeping interno no sustituye la verificación runtime. Excluye cambios de `KeyboardHook`, agregación de `unlockCore`, watchdog, Coordinator y persistencia de estado. Forecast: pequeño y claramente **≤400 líneas**.

**PR-05B1a2b** es exclusivamente `KeyboardHookWindows.Remove`, `unlockCore` y observabilidad watchdog. Propaga el resultado de `UnhookWindowsHookEx`, conserva estado active/handle veraz en fallo y lo limpia solo en éxito; `PostThreadMessageW` es limpieza best-effort y `threadID` permanece veraz. `unlockCore()` devuelve error, usa `errors.Join` para fallos aislados, continúa cleanup y `app_lifecycle` propaga el error físico. El watchdog elimina la remoción duplicada del hook y hace observable su fallo, sin retry y con exactamente una ruta física `EventUnlock`. Las pruebas cubren fallo native unhook, preservación/limpieza de active/handle, `PostThreadMessageW` best-effort, `threadID`, `unlockCore` total/ventana aislada/hook aislado/fallos unidos/continuidad de cleanup, watchdog sin remoción duplicada, error observable y sin `SessionActivation`; solo incluye la regresión de mantenimiento requerida por la propagación física de `OnEnter`. Excluye `SessionActivationCoordinator`, persistencia de estado de sesión, `station_login_enabled`, gates de `App.Unlock`/Dispatcher y recuperación PR-06. Debe permanecer dentro de la política normal **≤400 líneas**; no se comprimen pruebas para caber.

El contrato transversal de B1a1/B1a2a/B1a2b permanece explícito: SUCCESS físico exige tanto eliminación fullscreen confirmada como eliminación del keyboard hook confirmada; `UNKNOWN` crítico es FAILURE; las operaciones cosméticas/best-effort no hacen fallar automáticamente el desbloqueo. La FSM puede haber transitado `LOCKED → UNLOCKED` antes de un error de `OnEnter`/desbloqueo físico: el error se retorna, el destino permanece committed y no hay rollback automático. La recuperación de esa divergencia pertenece exclusivamente a PR-06.

**PR-05B1b queda SPLIT / SUPERSEDED y no es ejecutable. PR-05B1b1**, después de B1a2b y sobre la base `49ea661397cefd4fcd9ad07972ccca4bc91c71e2`, extiende exclusivamente `SessionRepository` con finalización pendiente: CAS obligatorio por `session_id` y `status='pending_unlock'`, exactamente una fila afectada y destinos finales limitados a `active` o `unlock_failed`. Aporta un sentinel mínimo de conflicto de transición y conflictos tipados mínimos de `PersistSessionActivation` para `session_id`, claim de estación y claim `(center_id,user_id)`, preservando causas útiles de la DB; no altera la atomicidad ni las semánticas de lectura existentes. **PR-05B1b2**, después de B1b1, hace a `SessionActivationCoordinator` la única ruta normal a `EventUnlock`: persiste `pending_unlock` antes del evento, lo emite a lo sumo una vez, finaliza éxito físico en `active` y fallo físico en `unlock_failed`, y conserva la ambigüedad de finalización con `errors.Join` cuando corresponda. B1b2 cubre replay, idempotencia y comparación de mismos datos; no incorpora cleanup, retry, recuperación ni cableado B2. **PR-05B2**, después de B1b2, incorpora `station_login_enabled`, las barreras de `App.Unlock` y de `UNLOCK` legado sin payload, compatibilidad con la función desactivada y regresiones de watchdog/recuperación y mantenimiento. `SyncIntent` conserva `aggregate_type=session`, versión `1` y estado `pending`; ningún sub-PR inventa semántica adicional.

RECOVERY/WATCHDOG sigue siendo una excepción privilegiada, identificable y auditable como `RECOVERY` con causa técnica, nunca `SessionActivation` ni datos sintéticos. B1a2b puede hacer observable su resultado físico sin cambiar su naturaleza. MAINTENANCE/ADMIN conserva la ruta separada `Maintenance authentication → MaintenanceUnlock → EventMaintenanceUnlock`; `unlockCore` compartido puede mejorar su observabilidad, sin alterar autenticación ni autorización.

### Nota operativa de referencia

El worktree de implementación de referencia es `C:/SisAIP/Soft_Usuario_PC/agente-infoplaza-pr-05b1a`. Contiene el GREEN parcial del B1a anterior y debe permanecer intacto. Su diff actual de 574 líneas contiene las 534 líneas originales de implementación más 40 líneas solo de formato introducidas por pi-lens. Cualquier transferencia futura será manual y por hunk: solo lógica funcional requerida, nunca cherry-pick del diff completo ni copia del ruido de formato de pi-lens.

### Temporizador

La fuente de verdad es la sesión (`started_at`, duración concedida, pausas/checkpoints y fin esperado), no un entero del renderer. El contador visible se deriva periódicamente. Reiniciar el proceso reconstruye el restante desde datos durables; cambios de estado relevantes se persisten, no cada tick. Umbrales de avisos y frecuencia de checkpoint son política configurable. Un salto de reloj que vuelva ambiguo el estado contiene la estación y crea caso de conciliación.

### Claves de conflicto e idempotencia (PR-05B1b1/B1b2 posteriores)

- Reintento de red: mismo `message_id` cuando es retransmisión del frame y mismo `idempotency_key` para la intención.
- Ante el mismo `session_id` y agregado idéntico, se consulta primero el agregado durable: si está `active`, puede devolverse éxito idempotente sin crear `Session`, claim ni `SyncIntent`, ni volver a desbloquear; si está `pending_unlock` o `unlock_failed`, se devuelve ese estado para recuperación, no éxito automático.
- El mismo `session_id` con datos distintos es `CONFLICT`, sin mutación. Un claim de estación o de `(center_id,user_id)` perteneciente a otra sesión también es `CONFLICT` y se conserva.
- Ante timeout o resultado desconocido, el llamador reutiliza el mismo `session_id` y consulta el estado durable; nunca asigna otro ID solo por timeout. La respuesta distingue no persistido, `pending_unlock`, `unlock_failed`, `active` y `CONFLICT`.
- Reinicio: consulta operación terminal por `request_id` y sesión por `session_id`; su recuperación pertenece a PR-06.
- Dinamizador: índices únicos sobre solicitud/idempotencia y sesión activa; inbox único por origen+hecho.
- Conciliación central futura: upsert por `fact_id/session_id`, no por documento, PC visible o timestamp.

## 11. PIN, bloqueo y recuperación

### Login y cambio

- Validación solo en backend; la UI recibe resultado genérico y, cuando aplique, tiempo de bloqueo permitido por política.
- Número de intentos, ventana, duración y coordinación entre estaciones son `PinAttemptPolicy`; no hay valores fijos.
- Con LAN, Dinamizador puede consolidar eventos por `user_id` y distribuir una marca de bloqueo versionada. Sin LAN se aplica la última marca válida y el control local.
- Un cambio o restablecimiento se completa mediante `CredentialReplacementService`: valida la prueba aplicable, confirma durablemente la nueva versión protegida y solo entonces invalida la anterior. El consumo de la temporal/grant y la activación nueva son atómicos en la autoridad que decide, o idempotentes con estado recuperable si cruzan autoridades. Los intercambios usan paquetes opacos protegidos, nunca PIN/hash en el protocolo de negocio.

### Flujos estables de recuperación y restablecimiento

Los nombres de modo son parte del contrato y de la auditoría: `EMAIL_RECOVERY`, `ASSISTED_STATION_RESET` y `DIRECT_DINAMIZADOR_RESET`. Comparten únicamente `CredentialReplacementService`, que activa una nueva `PermanentPinCredential` protegida y recién entonces invalida la anterior; no comparten credenciales temporales, grants ni supuestos de captura.

```text
EMAIL_RECOVERY
Usuario PC ──solicitud no secreta──> Plataforma central ──PIN temporal──> correo registrado
Usuario PC ──frontera protegida de validación/reemplazo──> autoridad de credencial
                                                    └── consume temporal + activa permanente

ASSISTED_STATION_RESET
Usuario PC ──solicitud/IDs──> Dinamizador ──decisión no secreta──> Usuario PC
                                  └── conserva grant server-side
Usuario en estación ──captura transitoria──> frontera protegida de reemplazo
                                              └── consume grant + activa permanente

DIRECT_DINAMIZADOR_RESET
Dinamizador ──autoriza──> grant server-side ──X──> ejecución/captura bloqueadas
                                             OPEN: actor y superficie de captura
```

Las flechas LAN/IPC y los eventos de aplicación transportan solo IDs, modo, estado y códigos no sensibles. El valor del grant permanece en backend; el PIN temporal solo cruza el adaptador central de entrega al correo. La frontera de validación/reemplazo es una interfaz de seguridad dedicada que no serializa PIN, verificador/hash, grant ni secreto en mensajes de negocio y queda fuera de logs, trazas y telemetría.

#### `EMAIL_RECOVERY`

1. Usuario PC solicita recuperación por `user_id`; no envía correo ni PIN al Dinamizador.
2. La plataforma central comprueba correo/canal, genera un `TemporaryEmailPinCredential` de cuatro dígitos y entrega el valor solamente al correo registrado. Nunca recupera ni envía el PIN permanente. Implementar el servicio de correo queda fuera de alcance; sí se define su contrato futuro de emisión, entrega, validación, reemplazo, expiración y consumo.
3. Una emisión nueva invalida la temporal activa anterior. La autoridad persiste, si es necesario, solo representación protegida y metadatos no secretos.
4. El usuario captura transitoriamente el PIN temporal y luego establece/confirma un PIN personal distinto conceptualmente. Solo una finalización satisfactoria consume la temporal y activa el nuevo verificador permanente; expiración, consumo previo o reemplazo dejan intacta la credencial permanente.
5. Sin correo, Internet, plataforma o canal disponible, el modo no produce credencial utilizable y la UI ofrece los modos con Dinamizador.

#### `ASSISTED_STATION_RESET`

1. Usuario PC origina una solicitud con IDs internos de usuario/solicitud/estación/centro, timestamp y documento solo si se necesita para identificación presencial; no incluye secretos.
2. Dinamizador valida presencialmente y aprueba o rechaza. La aprobación crea un `ResetAuthorizationGrant` de vigencia limitada, anti-replay, un uso y ligado a usuario+solicitud+estación+centro.
3. El usuario introduce y confirma el PIN nuevo en la estación vinculada; Dinamizador no conoce el anterior ni el nuevo.
4. La confirmación protegida reemplaza atómicamente la credencial permanente y consume el grant. Rechazo, expiración, cancelación, reemplazo, ámbito incorrecto o replay invalidan/rechazan el grant sin tocar la credencial vigente.

#### `DIRECT_DINAMIZADOR_RESET`

1. Un Dinamizador autorizado valida presencialmente e inicia el flujo administrativo desde su software, sin PIN anterior y sin solicitud originada en Usuario PC.
2. La autorización administrativa es temporal, anti-replay y de un uso, ligada al usuario, solicitud y centro; solo se liga a estación si la superficie finalmente aprobada lo requiere.
3. La credencial anterior permanece vigente hasta que la nueva quede establecida y confirmada satisfactoriamente; un flujo incompleto, fallido, cancelado o expirado no la invalida.
4. **OPEN REQUIREMENT bloqueante del subflujo:** quién captura el PIN nuevo y en qué superficie. El diseño no elige entre Usuario PC, Dinamizador u otra superficie. Hasta resolución, solo se implementan modelo, contratos, autorización y estados; la ejecución de reemplazo y toda UI de captura directa permanecen bloqueadas por feature gate.

## 12. Auditoría y minimización

- Pantalla pública: como máximo avatar, primer nombre, inicial del apellido, tiempo y costo cuando aplique. Documento completo, correo, dirección, nacimiento, detalle o indicador de discapacidad y datos administrativos no se muestran.
- Login fallido: tipo, tiempo, estación, resultado y `user_id` solo si fue resuelto; nunca PIN, verificador/hash, documento ni token de búsqueda.
- Recuperación/reset: lista permitida con modo estable, IDs de solicitud/usuario/solicitante/actor/autorizador, estación cuando aplique, centro, `occurred_at`, `received_at`, resultado y eventos de emisión, aprobación/rechazo, reemplazo, expiración, cancelación, consumo y finalización. El documento usado para validación presencial no se copia al evento.
- Discapacidad: los eventos de alta/cambio solo registran metadatos mínimos (`user_id`, actor, resultado, timestamp y que el indicador cambió si corresponde); no registran tipos seleccionados, etiquetas, texto libre ni payload completo.
- PIN actual, nuevo o temporal, verificadores/hashes, grants/autorizaciones, tokens, nonces y secretos están prohibidos en logs, auditoría, telemetría, mensajes de aplicación/protocolo, errores, trazas y diagnósticos, y también en UI administrativa innecesaria. No se serializan payloads completos.
- La UI administrativa solo proyecta metadatos y estados no secretos. Una futura superficie de captura aprobada será transitoria, no se reutilizará como vista administrativa y limpiará memoria/estado al enviar.
- Una función central de redacción y allowlists por evento se aplican antes de cualquier sink; pruebas con valores centinela verifican todos los sinks y adaptadores.
- Retención, acceso y exportación son políticas externas; no se introduce una retención fija nueva. Los timestamps de origen y recepción se conservan sin reescribir historia.

## 13. Política configurable

Se modela un documento versionado y validado, con precedencia central/Infoplaza/estación cuando esté definida, para:

- modalidad y habilitación del login;
- duración predeterminada y, si corresponde, duración aprobada;
- vigencia/frescura de usuario y credencial offline;
- máximo/ventana/bloqueo de intentos;
- heartbeat, reintentos, gracia y declaración de indisponibilidad LAN;
- timeout/cancelación de solicitud y expiración de reserva/grant;
- avisos/checkpoints del temporizador;
- vigencia de catálogos y retención/reintento de pendientes.

No se fijan días, intentos, duraciones de bloqueo ni umbrales de heartbeat en este diseño. El resolvedor clasifica cada parámetro y registra versión, procedencia y razón de fallback:

| Clase | Conducta ante ausencia/invalidez |
| --- | --- |
| Seguridad/autorización | Falla cerrada para la decisión protegida; no concede acceso ni privilegios. |
| Continuidad de sesión activa | Conserva el último estado y la última configuración válidos; no termina ni relockea una sesión válida por una actualización defectuosa. |
| Parámetro operativo no crítico | Usa la última configuración válida o un default seguro, versionado y documentado, sin fijar en código un valor de negocio variable. |
| Configuración necesaria para una operación nueva | Bloquea solamente esa operación y registra una razón estable/auditable; las demás capacidades y sesiones activas continúan. |

Por ejemplo, una política autónoma incompleta bloquea solo un nuevo inicio autónomo; una política de avisos inválida no invalida una sesión activa, y una autorización no verificable nunca se concede.

## 14. Impacto previsto por archivos

### Soft_Usuario_PC/agente-infoplaza

- **Conservar/ajustar:** `internal/lockdown/*`, `app_lock.go`, `frontend/src/components/PanelShell*`, `UserTabs*`.
- **Refactorizar:** `app.go`, `app_lifecycle.go`, `internal/connection/protocol.go`, `client.go`, `internal/commands/dispatcher.go`, `frontend/src/App.tsx`, `TimerTab.tsx`.
- **Extender:** `internal/config/config.go`, `config.example.json`, bindings Wails y documentación de persistencia/flujo/seguridad.
- **Agregar módulos enfocados:** `internal/persistence`, `internal/session`, `internal/auth`, `internal/catalog`, `internal/policy`, `internal/audit`, `internal/sync`; componentes de login/motivos/opciones/recuperación y sus hooks. `internal/auth` será dueño de `PermanentPinCredential`; un paquete `internal/recovery` separará contratos tipados de temporal/grant y gates por modo, sin adelantar su comportamiento en PR-04.

### Soft_Dinamizador

- **Extender:** `packages/shared-types/src/index.ts`, `electron/database/schema.ts`, `electron/database/repositories/pcs.repository.ts`, `electron/ipc/handlers.ts`, `electron/preload.ts`, `src/store/pcs.store.ts`, `PCsPage` y modal de asignación.
- **Follow-up de perfil/catálogos demográficos:** migraciones aditivas futuras para `usuarios.tiene_discapacidad`, `catalogo_discapacidades` y `usuario_discapacidades`; repositorios y UI de registro/edición con al menos una selección cuando `tiene_discapacidad=true`, sin decidir todavía si la cardinalidad será simple o múltiple, y sin logs/auditoría con valores.
- **Agregar módulos enfocados en main process:** gateway WSS/TLS 1.3 mTLS; adapters de certificado/registro/centro, lifecycle/clock/resumption, FSM/negociación/epoch-sequence y `SecurityAuditSink`; después, servicio de sesión, repositorios de autorización/auditoría/conciliación/motivos y adaptadores de outbox/inbox. Los repositorios de recuperación separarán solicitud, metadatos de temporal y grant server-side; ninguna clave privada o valor secreto será parte de IPC/preload/store.
- Los archivos existentes grandes no deben absorber toda la lógica; los handlers quedan como adaptadores finos.

### Plataforma central futura

Solo se publican contratos de IDs, hechos, catálogos, credenciales opacas y conflicto. Para `EMAIL_RECOVERY` se agrega el contrato futuro de generación/entrega/validación/ciclo de vida, no el servicio real de correo. No se agregan rutas de UI, roles detallados ni reportes.

## 15. Compatibilidad, despliegue y rollback

1. **Base sin activar:** migraciones aditivas, repositorios y telemetría de capacidades; flags apagados. Antes de migrar se valida backup recuperable de SQLite del Dinamizador.
2. **Secure link sin producto:** desplegar por slices A1/A2/B1/B2/U1/U2 WSS/TLS 1.3 mTLS, PKI por instalación, negociación `2/1`, `secure_link_v1`, epoch/sequence y auditoría durable. Conservar parsing v1 sin autorización; no habilitar handlers ni login. Verificar conformidad A+B+U antes de PR-11.
3. **Proyección sombra:** Usuario PC crea DB, recibe usuarios/catálogos/política y valida pendientes sin cambiar el flujo manual.
4. **Piloto automático/autorizado:** habilitación por estación/Infoplaza solo con participantes v2, vínculo autenticado y defensas anti-replay/duplicado para toda mutación privilegiada. Manual sigue visible.
5. **Autonomía:** se habilita separadamente después de configurar política, probar recuperación y aceptar el riesgo de unicidad durante aislamiento.
6. **Recuperación PIN:** `EMAIL_RECOVERY` y `ASSISTED_STATION_RESET` se habilitan por gates independientes solo con contratos, canal protegido, ciclos de vida y redacción probados. `DIRECT_DINAMIZADOR_RESET` conserva un gate adicional bloqueado: sus fundamentos pueden desplegarse, pero ejecución y UI de captura no se habilitan hasta resolver quién captura el PIN nuevo y dónde.

Rollback funcional desactiva `station_login_enabled`/capacidades v2 y deja la estación bloqueada a la espera del flujo manual sobre vínculo confiable. No baja esquema ni elimina sesiones, auditoría, facts o outbox. Dinamizador conserva lectura de columnas nuevas y comandos antiguos, pero no reactiva mutaciones LAN inseguras. Una versión antigua no debe abrir una DB migrada sin compatibilidad declarada; se revierte binario únicamente a una versión que tolere el esquema, o se restaura backup sin descartar facts posteriores.

Matriz esencial:

- nuevo Dinamizador + nueva estación: v2 autenticado y protegido contra replay/duplicados;
- nuevo Dinamizador + estación antigua: v1 manual solo mediante adaptador de vínculo confiable; sin él, actualización requerida para mutaciones remotas;
- Dinamizador antiguo + estación nueva: login v2 apagado; manual remoto solo si satisface las mismas garantías;
- fallo del núcleo o `SyncIntent`: rollback y estación bloqueada;
- fallo posterior aislable de fact/outbox/auditoría de negocio: sesión habilitable, intención conservada y materialización/reintento idempotente; fallo del `SecurityAuditSink` obligatorio mantiene falsa la elegibilidad privilegiada.

## 16. Estrategia de pruebas para TDD estricto posterior

No se ejecutan pruebas en esta fase. Cada incremento de implementación deberá comenzar por una prueba fallida.

### Usuario PC — unitarias Go

- normalización/token de documento sin persistir PII;
- `PermanentPinCredential` válido/inválido, verificador no reversible protegido y redacción;
- separación de tipos: una `TemporaryEmailPinCredential` o un `ResetAuthorizationGrant` nunca autentican login ordinario;
- clasificación de políticas: seguridad fail-closed, continuidad con último válido, operación no crítica con fallback seguro y bloqueo localizado de operaciones nuevas;
- decisiones automática/autorizada/autónoma, incluida pérdida solo de Internet y ausencia selectiva de configuración;
- solicitud terminal, rechazo, cancelación y respuesta repetida/tardía;
- PR-05B1a1: `StateEnterCallback`/`OnEnter` error-capable, `from`/`to`, `errors.Is`, callback exactamente una vez, destino committed y sin rollback; transición inválida sin callback;
- PR-05B1a2a: fullscreen false confirmado, fullscreen true y `UNKNOWN` críticos como FAILURE, cosméticos best-effort y bookkeeping incapaz de sustituir verificación runtime;
- PR-05B1a2b: fallo native unhook, estado active/handle y `threadID` veraces, `PostThreadMessageW` best-effort, `errors.Join`, cleanup continuo, una ruta física watchdog sin remoción duplicada/error observable/retry/`SessionActivation`;
- PR-05B1b1: finalización solo desde `pending_unlock` por CAS de `session_id` y estado, exactamente una fila, destinos `active`/`unlock_failed`, sentinel mínimo de conflicto y causas DB útiles para conflictos de sesión, claim de estación y claim `(center_id,user_id)`;
  - PR-05B1b2: `pending_unlock` antes de `EventUnlock`, evento a lo sumo una vez, `active` solo tras éxito físico B1a2b y `unlock_failed` tras fallo; idempotencia, replay, comparación de mismos datos y fallo de finalización como ambigüedad sin cleanup, retry ni recuperación automática;
- PR-05B2: `App.Unlock` sin bypass ni datos sintéticos, `UNLOCK` legado sin payload bloqueado con login habilitado y compatibilidad feature-OFF; regresiones de watchdog/recuperación y mantenimiento preservadas;
- PR-06: recuperación tras reinicio, temporizador durable, reconstrucción de inicio y resolución durable de `pending_unlock`, `unlock_failed` o divergencias ambiguas.
- `EMAIL_RECOVERY`: emisión central contractual de cuatro dígitos, expiración, un uso, reemplazo de la emisión anterior y cambio permanente solo al finalizar;
- `ASSISTED_STATION_RESET`: grant ligado a usuario/solicitud/estación, rechazo de replay/ámbito incorrecto y credencial anterior vigente ante fallo;
- `DIRECT_DINAMIZADOR_RESET`: autorización sin PIN anterior y bloqueo de ejecución/captura mientras siga abierto el actor/surface.

### Usuario PC — integración Go/SQLite/WebSocket

- migración desde instalación sin DB usando `t.TempDir()`;
- transacción del núcleo+`SyncIntent` con `pending_unlock`, rollback por fallo nuclear y commit seguido de `EventUnlock`; éxito físico seguido de actualización a `active`, fallo físico con intento de `unlock_failed` y fallo de actualización como ambigüedad conservada;
- materialización idempotente de fact/outbox/auditoría tras reinicio y alerta ante fallo persistente;
- constraints de sesión activa, reapertura de DB y supervivencia de pendientes;
- snapshots históricos después de cambiar catálogo;
- PR-07: discriminador sin downgrade, codec bidireccional v1/v2, validación exacta del sobre, round-trip de `response`/`ack`, callback v2 inerte, dedupe FIFO por `message_id`, continuidad del read loop y redacción de logs;
- PR-08A1/A2/B1/B2/U1/U2: mTLS y binding de instalación/centro, schemas exactos `link.hello|accept|reject`, capability `secure_link_v1`, época no cero, secuencia direccional exacta, reconnect, reloj/expiración, deny local, rechazo por canal y fallo del `SecurityAuditSink`, siempre con cero acciones de producto;
- desconexión/reconexión con reenvío del mismo fact y sin duplicar sesión;
- verificación con secretos centinela de que DB no autorizada, logs, auditoría, IPC, LAN, telemetría, errores y trazas no contienen PIN/verificador/hash/grant/token; el único canal que recibe el PIN temporal es el adaptador central de entrega de correo.

### Usuario PC — frontend Vitest

- documento/PIN requerido y PIN de cuatro dígitos;
- credenciales rechazadas con mensaje no enumerativo;
- selección múltiple y bloqueo sin motivos;
- filtros por fecha/Infoplaza/estado y continuidad sin opciones;
- espera autorizada, rechazo, cancelación, estado offline/autónomo;
- recuperación por correo no disponible, captura transitoria de PIN temporal y solicitud asistida;
- ausencia de UI ejecutable para `DIRECT_DINAMIZADOR_RESET` mientras siga abierto quién captura y dónde;
- widget con datos mínimos reales, sin PII y sin valores demo.

### Dinamizador

Primero se establece un runner de pruebas compatible con TypeScript/Electron. Luego:

- unitarias futuras de catálogo/perfil de discapacidad: catálogo configurable, `TRUE/FALSE/NULL` con semántica legacy, al menos una selección requerida cuando `tiene_discapacidad=true`, limpieza de asociaciones activas cuando es falso, persistencia normalizada y redacción de valores sensibles;
- unitarias del servicio único de sesión para flujo manual/LAN, reserva, activación, transferencia e idempotencia;
- integración con SQLite temporal para migraciones, índices únicos, núcleo+`SyncIntent` atómicos y materialización posterior de outbox/conciliación;
- handlers IPC y LAN como adaptadores que no divergen en reglas;
- aprobación/rechazo y reset sin exponer grants o credenciales al renderer;
- ciclo independiente de temporal por correo, grant asistido ligado a estación y autorización directa con estación opcional;
- compatibilidad del gateway con clientes v1 y v2;
- conflicto explícito en lugar de `dashboard-wins` para hechos de sesión.

### E2E-ish entre componentes

Con gateway de prueba, dos procesos/servicios y DBs temporales:

- login automático completo y barrera de durabilidad;
- autorizado con respuesta perdida y replay;
- autónomo tras umbral de política simulado, reinicio y conciliación;
- dos estaciones intentando el mismo usuario, tanto con LAN como aisladas;
- Dinamizador reiniciado con reserva y estación con sesión durable;
- flujo manual en combinación de versiones, rechazando toda mutación sin vínculo autenticado o con replay;
- recuperación por correo con temporal reemplazada/expirada/consumida y servicio de entrega simulado;
- reset asistido de un solo uso y resistente a replay;
- reset directo limitado a fundamentos/contratos, demostrando que su ejecución y captura siguen bloqueadas;
- fallo de disco, frame corrupto, respuesta tardía y reloj alterado.

## 17. Riesgos y tratamientos aprobados

| Riesgo | Tratamiento de diseño |
| --- | --- |
| Dinamizador no posee actualmente gateway LAN | Se agrega en main process y se prueba antes de habilitar login; el flujo manual contiene el rollout. |
| WebSocket actual no autentica servidor ni protege transporte | Migrar el camino confiable a WSS/TLS 1.3 mTLS por instalación, con binding a centro, negociación `2/1`, epoch/secuencia y deny local; WS en claro y legado no autenticado otorgan cero privilegio. |
| Unicidad global durante aislamiento de varias estaciones | Invariante futuro acotado: los duplicados offline no se convierten silenciosamente en identidades independientes; se registran hechos locales y posteriormente se concilian o se marcan conflictos cuando corresponda. Este cambio no diseña el comportamiento de conciliación. |
| Nueva SQLite en Usuario PC aumenta superficie operativa | Base única embebida, repositorios pequeños, migraciones, backup/recuperación y driver validado para Windows. |
| Dos cronómetros y dos interpretaciones de sesión | Sesión durable única; ambos timers son proyecciones y todos los orígenes usan el mismo servicio/ID. |
| Credenciales, discapacidad y PII en estaciones públicas | Proyección mínima, token de búsqueda, DPAPI, paquetes opacos, ACL, redacción, no exposición pública de discapacidad y canal seguro. |
| Esquema actual y sync no son atómicos para sesiones | Migración aditiva; núcleo de sesión + `SyncIntent` atómicos, con fact/outbox/auditoría recuperables e idempotentes fuera de la barrera. |
| Política aún sin valores exactos | Documento versionado, clasificación por criticidad y fallback localizado; ninguna constante fija de negocio. |

## 18. Decisiones aprobadas y preparación para tareas

Las decisiones de revisión quedan aprobadas con estas precisiones vinculantes:

1. PR-05B1a y PR-05B1b están **SPLIT / SUPERSEDED** y no tienen superficie ejecutable propia: la cadena aprobada es PR-05A1 → PR-05A2 → **PR-05B1a1 (propagación FSM `OnEnter`)** → **PR-05B1a2a (postcondición de ventana)** → **PR-05B1a2b (hook, `unlockCore` y watchdog)** → **PR-05B1b1 (transiciones de repositorio y semántica de conflicto)** → **PR-05B1b2 (coordinador y barrera física)** → **PR-05B2 (barreras legadas y función/configuración)** → **PR-06 (recuperación durable y temporizador)** → **PR-07 (sobre v2 y parser legado seguro)**. Los detalles ejecutables constan en `tasks.md`.
2. B1a1 entrega solo la firma/error semántico de FSM y el puente `app_lifecycle.go` que retorna `nil`; no propaga todavía error físico. B1a2a confirma la eliminación fullscreen antes de bookkeeping y trata `UNKNOWN` crítico como FAILURE. B1a2b confirma la eliminación del hook, propaga el error físico desde `unlockCore` y hace observable el watchdog.
3. SUCCESS físico exige ambas confirmaciones críticas —fullscreen removido y keyboard hook removido—; `UNKNOWN != SUCCESS`. Las operaciones cosméticas/best-effort no hacen fallar por sí solas el desbloqueo. La FSM no ejecuta rollback: si `LOCKED → UNLOCKED` precede a un error, retorna y reporta la divergencia aunque permanezca `UNLOCKED`.
4. B1a2a/B1a2b se limitan a resultados nativos y comprobaciones Windows mínimas; preservan `FocusEnforcer`, WATCHDOG/RECOVERY y MAINTENANCE/ADMIN. No autorizan Coordinator, persistencia de sesión, gates, segunda FSM, lockdown duplicado, marco Win32 general, retries, recuperación ni transporte.
5. PR-05B1b1 es exclusivamente la finalización pendiente de `SessionRepository` por CAS y sus conflictos tipados; PR-05B1b2 es exclusivamente `SessionActivationCoordinator`, replay/idempotencia/comparación de mismos datos y la barrera física; PR-05B2 es exclusivamente barreras legadas más feature/config plumbing; PR-06 es el único dueño de reconstrucción al inicio, temporizador durable y reconciliación durable de estados o físico ambiguos después de reiniciar.
6. `SyncIntent` conserva `aggregate_type=session`, versión `1` y estado `pending`; la futura sincronización de cambios de estado es una dependencia no bloqueante.
7. Usuario PC incorpora SQLite embebido y protección local; Dinamizador extiende el único modelo `sesiones`.
8. Toda mutación LAN privilegiada futura exige WSS/TLS 1.3 mTLS, instalación y centro autorizados, `2/1`, `secure_link_v1`, epoch/secuencia frescos, allowlist de operación y autorización posterior; PR-08 habilita cero acciones. La unicidad de sesión durante aislamiento sigue usando prevención conocida y conflicto explícito, sin consenso inventado.
9. Las políticas se clasifican por seguridad, continuidad, criticidad operativa y necesidad para operaciones nuevas; el rollout es gradual, reversible y sujeto a TDD estricto.
10. `PermanentPinCredential`, `TemporaryEmailPinCredential` y `ResetAuthorizationGrant` son tipos incompatibles con repositorios y ciclos de vida separados. La plataforma central futura es dueña del contrato de generación/entrega de `EMAIL_RECOVERY`; su servicio real de correo no forma parte de este cambio.
11. `ASSISTED_STATION_RESET` obliga a captura del PIN nuevo por el usuario en la estación; `DIRECT_DINAMIZADOR_RESET` no hereda ese supuesto.

### Interacción vinculante con el plan de tareas

- **PR-04** puede implementar únicamente fundamentos de `PermanentPinCredential` para autenticación ordinaria y seams/interfaces tipadas para distinguir credencial temporal y grant. No puede generar, persistir, validar, consumir ni invalidar una temporal de correo o un grant de reset. **PR-04C conserva íntegramente su estado BLOCKED / DEFERRED PENDING OPEN REQUIREMENTS y todos sus límites de política; no se vuelve prerrequisito de PR-05A1, PR-05A2 ni PR-05B1a1.**
- **PR-05A1** implementa únicamente migración v3 y el esquema de persistencia; **PR-05A2** implementa únicamente adquisición atómica del claim con `Session`, motivos y `SyncIntent`, rechazo de duplicados, rollback, integridad referencial y prevención de huérfanos, sin liberación. **PR-05B1a1** implementa solo callback/FSM error-capable y el puente `app_lifecycle.go`; **PR-05B1a2a**, solo postcondición de ventana; **PR-05B1a2b**, solo hook/`unlockCore`/watchdog observable. **PR-05B1b1** extiende exclusivamente `SessionRepository` con finalización `pending_unlock` por CAS de una fila hacia `active`/`unlock_failed` y conflictos tipados que conservan causas DB; **PR-05B1b2** consume ese resultado para `SessionActivationCoordinator`, persistencia previa `pending_unlock`, un `EventUnlock` a lo sumo una vez, finalización, idempotencia, replay y comparación de mismos datos. **PR-05B2** incorpora exclusivamente función/configuración y barreras `App.Unlock`/dispatcher legado. Ninguno altera el contrato inicial de `SyncIntent`, libera claims ni adelanta cleanup, retry o recuperación.
- **PR-06** conserva sin adelantos su alcance de recuperación tras reinicio, proyección durable del temporizador, reconstrucción de inicio y conciliación de `pending_unlock`, `unlock_failed` y estados ambiguos; PR-05A1/PR-05A2/PR-05B1a1/PR-05B1a2a/PR-05B1a2b/PR-05B1b1/PR-05B1b2/PR-05B2 no absorben ese comportamiento.
- **PR-15** implementará fundamentos y contratos de los tres modos, incluida la interfaz central futura de `EMAIL_RECOVERY`, ciclos de vida de temporal/grant, reemplazo seguro y auditoría redactada. La ejecución y cualquier UI de captura de `DIRECT_DINAMIZADOR_RESET` deben quedar bloqueadas por gate hasta resolver el OPEN REQUIREMENT.
- **PR-01** no toca modelos ni flujos de PIN: su spike de driver SQLite puro permanece sin cambios y apply-ready.

Dentro de la revisión de PIN, el único bloqueo funcional restante es quién captura el PIN nuevo y en qué superficie para `DIRECT_DINAMIZADOR_RESET`; bloquea solo su ejecución/captura, no PR-01, los fundamentos/contratos de PR-15 ni PR-05B1a1. Separadamente, los nueve bloqueadores técnicos PR-08 están CLOSED y listos para planificación, pero no autorizan `apply`: los umbrellas A/B/U son no ejecutables y el primer worktree potencial es A1 después de un preflight nuevo. No queda un bloqueo funcional nuevo para los preflights B1a1/B1a2a/B1a2b. OpenSpec vive deliberadamente fuera del HEAD Git de Usuario PC; su ausencia dentro de ese repositorio no bloquea esta cadena documental ni su preflight.
