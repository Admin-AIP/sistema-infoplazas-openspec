# Especificación de acceso e inicio de sesión de Usuario PC

## Purpose

Definir el acceso de usuarios registrados desde una estación de Usuario PC, incluyendo autenticación, motivos de visita, modalidades de inicio, continuidad offline autorizada, protección y recuperación del PIN, durabilidad, conflictos de sesión, auditoría y conciliación con el Dinamizador y la autoridad central.

## Requirements

### Requirement: Identificación y autenticación del usuario

El sistema MUST solicitar cédula o pasaporte como criterio documental de búsqueda y un PIN de exactamente cuatro dígitos creado por el usuario. El sistema MUST asociar el acceso y la sesión a identificadores internos estables, y MUST NOT usar el documento como clave técnica primaria. El sistema MUST rechazar entradas incompletas, un PIN que no conste de cuatro dígitos o credenciales inválidas sin habilitar la estación.

#### Scenario: Credenciales válidas

- GIVEN un usuario registrado, activo y elegible para usar la estación
- AND el usuario tiene una credencial local válida
- WHEN ingresa su cédula o pasaporte y su PIN correcto de cuatro dígitos
- THEN el sistema MUST autenticar al usuario contra su identidad interna estable
- AND MUST permitir que continúe hacia la declaración de motivos

#### Scenario: Entrada inválida o credencial incorrecta

- GIVEN una estación bloqueada sin sesión activa
- WHEN el usuario omite el documento, ingresa un PIN que no tiene cuatro dígitos o presenta una credencial incorrecta
- THEN el sistema MUST rechazar el intento
- AND MUST mantener bloqueada la estación
- AND MUST registrar el resultado fallido sin incluir el PIN

### Requirement: Identidad nacional y alcance operativo

Para ADA Nova Plus, la identidad del usuario MUST ser nacionalmente global: `user_id` es la identidad global estable. `center_scope` MUST ser obligatorio para el alcance operativo, de elegibilidad y de proyección local, pero MUST NOT formar parte de la identidad del usuario. La credencial PIN permanente MUST pertenecer al usuario global, no a un `center_scope`. Como consecuencia de esquema de PR-04A, `user_projections` MUST usar `PRIMARY KEY(user_id)`; `permanent_pin_credentials.user_id` MUST declarar `FOREIGN KEY(user_id) REFERENCES user_projections(user_id)`; y `UNIQUE(center_scope, document_lookup_token)` MUST conservarse como unicidad de búsqueda operativa sin convertir `center_scope` en componente de identidad.

#### Scenario: Misma identidad global en un alcance operativo

- GIVEN una proyección de usuario de ADA Nova Plus para un `center_scope` obligatorio
- WHEN el sistema identifica, proyecta o valida su credencial permanente
- THEN MUST usar el mismo `user_id` global estable como identidad y vínculo de credencial
- AND MUST aplicar `center_scope` solo para el alcance operativo, elegibilidad o proyección local
- AND MUST NOT crear una identidad ni una credencial PIN permanente independientes por `center_scope`

### Requirement: Discapacidad en el perfil del usuario

El sistema MUST registrar en el perfil del usuario si tiene alguna discapacidad mediante `usuarios.tiene_discapacidad` con semántica nullable: `TRUE` significa que el usuario indicó que sí, `FALSE` significa que indicó que no, y `NULL` significa dato todavía no capturado para usuarios legados o perfiles incompletos. `NULL` MUST NOT interpretarse como `FALSE`. Cuando el valor sea afirmativo, el sistema MUST presentar un catálogo configurable de tipos de discapacidad y MUST exigir al menos una selección válida según la cardinalidad que se apruebe. Los tipos seleccionados MUST persistirse como relaciones normalizadas (`usuario_discapacidades`) contra un catálogo configurable (`catalogo_discapacidades`), y MUST NOT guardarse como texto libre dentro de `usuarios` ni codificarse de forma fija en la UI o el backend. Esta información MUST tratarse como dato personal sensible: no debe aparecer en logs, auditoría con valores, telemetría, pantallas públicas ni superficies administrativas que no la necesiten.

#### Scenario: Usuario sin discapacidad declarada

- GIVEN que se registra o actualiza el perfil de un usuario
- WHEN el operador marca que el usuario no tiene discapacidad
- THEN el sistema MUST guardar `tiene_discapacidad` como falso o equivalente
- AND MUST NOT exigir tipos de discapacidad asociados
- AND MUST NOT conservar asociaciones activas para ese perfil

#### Scenario: Usuario legado sin dato capturado

- GIVEN un usuario existente cuyo dato de discapacidad todavía no fue capturado
- WHEN el sistema lee o sincroniza su perfil
- THEN MUST conservar `tiene_discapacidad` como `NULL` o equivalente desconocido
- AND MUST NOT tratarlo como usuario sin discapacidad
- AND MUST NOT inventar asociaciones de discapacidad

#### Scenario: Usuario con discapacidad declarada

- GIVEN que existe un catálogo válido de tipos de discapacidad activos
- WHEN el operador marca que el usuario tiene discapacidad
- THEN el sistema MUST mostrar opciones provenientes del catálogo configurable
- AND MUST exigir al menos una selección válida antes de guardar

#### Scenario: Catálogo configurable y normalizado

- GIVEN que el catálogo de discapacidades cambia, desactiva o agrega opciones
- WHEN se registra o actualiza el perfil de un usuario
- THEN el sistema MUST usar IDs de catálogo vigentes para las nuevas selecciones
- AND MUST NOT guardar el tipo de discapacidad como texto libre dentro de `usuarios`
- AND MUST preservar las relaciones históricas existentes por ID sin reescribirlas como texto libre

#### Scenario: Protección de dato sensible de discapacidad

- GIVEN que un perfil contiene información de discapacidad
- WHEN el sistema registra logs, auditoría, telemetría, errores, diagnósticos o pantallas públicas
- THEN MUST omit disability type values and unnecessary disability details
- AND MAY include only minimal internal identifiers or change/result metadata when strictly required for traceability

### Requirement: Motivos de visita y opciones programadas

El sistema MUST exigir la selección de uno o más motivos desde el último catálogo válido disponible. La selección MUST admitir múltiples motivos. Cuando los motivos incluyan “Capacitación presencial”, “Capacitación en línea” o sus equivalentes catalogados, el sistema MUST mostrar las capacitaciones planificadas para la fecha y la Infoplaza. Cuando incluyan “Participar en una actividad” o su equivalente catalogado, MUST mostrar las actividades programadas para la fecha y la Infoplaza. El sistema MUST NOT ofrecer como seleccionables opciones inactivas, canceladas, de otra fecha o de otra Infoplaza. La ausencia de opciones programadas MUST NOT impedir el inicio de sesión. Los valores de motivos y opciones seleccionados para una sesión MUST permanecer históricamente estables aunque el catálogo cambie posteriormente.

#### Scenario: Selección de varios motivos

- GIVEN que el catálogo local válido contiene varios motivos activos
- WHEN el usuario selecciona dos o más motivos
- THEN el sistema MUST conservar todas las selecciones para la sesión
- AND MUST permitir continuar si las demás validaciones son satisfactorias

#### Scenario: Motivo que presenta capacitaciones

- GIVEN que el usuario seleccionó un motivo de capacitación
- WHEN existen capacitaciones activas planificadas para esa fecha e Infoplaza
- THEN el sistema MUST mostrar solamente las opciones que coinciden con ambas condiciones
- AND MUST permitir registrar la opción o las opciones que el usuario seleccione

#### Scenario: Motivo que presenta actividades

- GIVEN que el usuario seleccionó el motivo de participar en una actividad
- WHEN existen actividades activas programadas para esa fecha e Infoplaza
- THEN el sistema MUST mostrar solamente las opciones que coinciden con ambas condiciones
- AND MUST NOT permitir seleccionar una opción inactiva o cancelada

#### Scenario: No hay opciones programadas

- GIVEN que un motivo seleccionado requiere consultar capacitaciones o actividades
- AND no hay opciones programadas válidas para esa fecha e Infoplaza
- WHEN el usuario continúa el flujo
- THEN el sistema MUST informar que no existen opciones disponibles
- AND MUST permitir continuar sin una selección programada

#### Scenario: Falta un motivo

- GIVEN que las credenciales del usuario son válidas
- WHEN intenta continuar sin seleccionar al menos un motivo
- THEN el sistema MUST impedir el inicio de sesión
- AND MUST solicitar al menos una selección válida

#### Scenario: El catálogo cambia después de una sesión

- GIVEN una sesión que registró motivos y opciones programadas seleccionadas
- WHEN el catálogo modifica, desactiva o elimina posteriormente esos valores
- THEN el historial de la sesión MUST conservar los valores seleccionados originalmente
- AND MUST NOT reinterpretarlos usando el catálogo nuevo

### Requirement: Modalidad automática

Cuando la Infoplaza o estación esté configurada en modalidad automática, el sistema MUST iniciar la sesión de un usuario autenticado y elegible sin exigir aprobación individual del Dinamizador. La sesión MUST registrar explícitamente la modalidad automática y MUST usar el tiempo predeterminado válido configurado.

#### Scenario: Inicio automático satisfactorio

- GIVEN una estación configurada para inicio automático
- AND un usuario autenticado, elegible y con motivos válidos
- WHEN finaliza las validaciones de acceso
- THEN el sistema MUST crear una sesión con modalidad automática
- AND MUST asignar el tiempo predeterminado válido
- AND MUST NOT exigir una aprobación individual del Dinamizador

### Requirement: Modalidad autorizada por Dinamizador

Cuando la Infoplaza o estación esté configurada para inicio autorizado, el sistema MUST solicitar una decisión al Dinamizador y MUST mantener bloqueada la estación hasta recibir y aplicar una aprobación válida. Un rechazo, cancelación o falta de aprobación MUST NOT crear ni habilitar una sesión. Una respuesta repetida o tardía MUST NOT crear una segunda sesión.

#### Scenario: El Dinamizador aprueba la solicitud

- GIVEN un usuario autenticado y elegible en una estación configurada para inicio autorizado
- AND el usuario completó los motivos requeridos
- WHEN el Dinamizador aprueba la solicitud vigente
- THEN el sistema MUST crear una sesión con modalidad autorizada
- AND MUST aplicar el tiempo aprobado o el tiempo predeterminado válido que corresponda

#### Scenario: El Dinamizador rechaza la solicitud

- GIVEN una solicitud vigente de inicio autorizado
- WHEN el Dinamizador la rechaza
- THEN el sistema MUST informar el rechazo sin habilitar la estación
- AND MUST NOT crear una sesión activa

#### Scenario: Llega una aprobación repetida o tardía

- GIVEN que la solicitud ya fue completada, cancelada o reemplazada
- WHEN llega una aprobación repetida o tardía
- THEN el sistema MUST NOT crear una sesión adicional
- AND MUST conservar un resultado coherente y auditable

### Requirement: Activación y comportamiento de modalidad autónoma

La modalidad autónoma MUST estar disponible únicamente cuando la Infoplaza esté configurada para inicio autorizado y Usuario PC no pueda comunicarse temporalmente con el Software de Administración del Dinamizador. La pérdida de Internet por sí sola MUST NOT activar la modalidad autónoma mientras la comunicación LAN con el Dinamizador continúe disponible. El sistema MUST aplicar comprobaciones técnicas y reintentos antes de considerar interrumpida la comunicación, de forma que una interrupción mínima o transitoria no active esta modalidad. En modalidad autónoma, el sistema MUST usar el último tiempo predeterminado válido, los últimos catálogos sincronizados, los últimos motivos disponibles y la última configuración válida. Cada sesión así iniciada MUST identificarse como autónoma, registrarse localmente y quedar pendiente de conciliación posterior.

#### Scenario: Falla temporal de comunicación con el Dinamizador

- GIVEN una Infoplaza configurada para inicio autorizado
- AND Usuario PC no puede comunicarse temporalmente con el Dinamizador después de las comprobaciones y reintentos aplicables
- AND el usuario cumple la elegibilidad offline
- WHEN el usuario completa credenciales y motivos válidos
- THEN el sistema MUST permitir el inicio conforme a la política autónoma autorizada
- AND MUST registrar la sesión con modalidad autónoma y el último tiempo predeterminado válido
- AND MUST marcarla para conciliación posterior

#### Scenario: Solo se pierde Internet

- GIVEN una Infoplaza configurada para inicio autorizado
- AND no hay conectividad a Internet
- BUT Usuario PC conserva comunicación LAN con el Dinamizador
- WHEN el usuario solicita iniciar sesión
- THEN el sistema MUST mantener el flujo autorizado con el Dinamizador
- AND MUST NOT activar modalidad autónoma por la sola pérdida de Internet

#### Scenario: Interrupción mínima o transitoria

- GIVEN que se produce una interrupción breve de comunicación con el Dinamizador
- WHEN las comprobaciones o reintentos recuperan la comunicación antes del umbral configurado
- THEN el sistema MUST continuar o recuperar el flujo autorizado
- AND MUST NOT iniciar una sesión autónoma

### Requirement: Elegibilidad para autenticación offline

Para autenticar sin comunicación disponible con el Dinamizador, el sistema MUST comprobar que el usuario existe localmente, está activo, posee una credencial local válida y no tiene una marca local de bloqueo o deshabilitación. Si alguna condición falla o no puede comprobarse con datos locales válidos, el sistema MUST negar el inicio autónomo. La ausencia de una política aprobada de antigüedad de credencial MUST NOT dar lugar a un límite de días codificado de forma fija.

#### Scenario: Usuario elegible offline

- GIVEN que el usuario existe en los datos locales
- AND figura activo y sin marca local de bloqueo o deshabilitación
- AND su credencial local es válida
- WHEN solicita acceso durante una interrupción que permite modalidad autónoma
- THEN el sistema MUST considerarlo elegible para continuar con las demás validaciones

#### Scenario: Usuario inelegible offline

- GIVEN que el usuario no existe localmente, está inactivo, está marcado como bloqueado o deshabilitado, o carece de credencial local válida
- WHEN solicita acceso sin comunicación con el Dinamizador
- THEN el sistema MUST negar el inicio autónomo
- AND MUST mantener bloqueada la estación

### Requirement: Unicidad, recuperación y transferencia de sesiones activas

Una estación con una sesión activa MUST NOT iniciar una segunda sesión simultánea. Un usuario MUST NOT tener más de una sesión activa en la misma Infoplaza. Ante un intento duplicado, el sistema MUST localizar la sesión existente y MUST ofrecer recuperación o continuación cuando corresponda. Si el usuario cambia de PC, el cambio MUST realizarse como transferencia de la misma sesión, preservando su identidad, usuario, tiempo, motivos, datos de cobro aplicables y trazabilidad de las PCs; MUST NOT crear una sesión nueva.

#### Scenario: La estación ya tiene una sesión activa

- GIVEN una estación con una sesión activa
- WHEN se intenta iniciar otra sesión en esa estación
- THEN el sistema MUST rechazar la creación de una segunda sesión simultánea
- AND MUST mantener consistente la sesión existente

#### Scenario: El usuario ya tiene una sesión en la misma Infoplaza

- GIVEN un usuario con una sesión activa en la misma Infoplaza
- WHEN intenta iniciar otra sesión
- THEN el sistema MUST NOT crear una segunda sesión
- AND MUST localizar la sesión existente
- AND MUST permitir su recuperación o continuación según el estado vigente

#### Scenario: El usuario cambia de estación

- GIVEN un usuario con una sesión activa en una estación de la Infoplaza
- WHEN se aprueba su cambio a otra estación
- THEN el sistema MUST transferir la misma sesión
- AND MUST preservar usuario, tiempo, motivos y datos de cobro aplicables
- AND MUST conservar trazabilidad de la estación de origen y la de destino

### Requirement: Durabilidad antes de habilitar la estación

El sistema MUST confirmar de forma durable el registro crítico de la sesión, su modalidad, usuario, estación, tiempo, motivos, `SessionBlockingClaim` y `SyncIntent` antes de desbloquear o considerar habilitada la estación. `PersistSessionActivation` MUST registrar el agregado nuevo exactamente con `Session.status = pending_unlock`: el claim y la intención existen duraderamente, pero la estación todavía no está físicamente activa. El sistema MUST NOT persistir `active` antes de confirmar el desbloqueo físico. `SyncIntent` MUST conservar el contrato inicial `aggregate_type=session`, versión `1` y estado `pending`; los estados de `Session` MUST NOT inventar semántica nueva para esa intención. Los reintentos, reinicios y respuestas repetidas MUST NOT crear sesiones duplicadas. Si la persistencia falla o el estado queda ambiguo, el sistema MUST mantener la estación bloqueada y conservar o presentar el caso para recuperación. Los registros pendientes MUST sobrevivir desconexiones y reinicios hasta poder sincronizarse o conciliarse.

#### Scenario: Persistencia confirmada antes del desbloqueo físico

- GIVEN un inicio de sesión aprobado por la modalidad aplicable
- WHEN `PersistSessionActivation` confirma durablemente el agregado crítico
- THEN `Session.status` MUST ser exactamente `pending_unlock`
- AND MUST existir duraderamente la sesión, sus motivos, el claim y el `SyncIntent`
- AND la estación MUST seguir bloqueada hasta el éxito físico confirmado

#### Scenario: Falla la persistencia local

- GIVEN un inicio de sesión que superó autenticación y autorización
- WHEN no puede confirmarse el registro durable de la sesión
- THEN el sistema MUST mantener bloqueada la estación
- AND MUST registrar el fallo sin afirmar que existe una sesión habilitada

#### Scenario: Reinicio con una operación pendiente

- GIVEN una sesión local o transacción pendiente de sincronización
- WHEN Usuario PC se reinicia antes de conciliarla
- THEN el sistema MUST recuperar el registro pendiente
- AND MUST reintentar o presentar su conciliación sin duplicar la sesión

### Requirement: Subdivisión aprobada de PR-05B1a

**PR-05B1a está SPLIT / SUPERSEDED**. La cadena obligatoria es PR-05B1a1 (FSM OnEnter error propagation) → PR-05B1a2a (Window unlock postcondition observability) → PR-05B1a2b (Keyboard hook + unlockCore + watchdog observability) → PR-05B1b (`SessionActivationCoordinator`) → PR-05B2 (legacy barriers + feature/config plumbing) → PR-06 (restart recovery + durable timer). Cada unidad MUST respetar la política normal de ≤400 líneas; 401–450 requiere excepción explícita de cohesión y >450 MUST detenerse para una nueva subdivisión, sin comprimir pruebas artificialmente.

### Requirement: Propagación de errores FSM `OnEnter` (PR-05B1a1)

`StateEnterCallback` MUST retornar `error` y `OnEnter` MUST ser error-capable. Para una transición válida, la FSM MUST entregar los valores `from` y `to` correctos, invocar el callback exactamente una vez y devolver desde `Send` el error de `OnEnter` preservando `errors.Is`. El estado destino MUST permanecer committed aun cuando `OnEnter` falle; el sistema MUST NOT ejecutar rollback automático. Una transición inválida MUST omitir el callback. El único cambio de aplicación permitido es el puente mínimo de compilación en `app_lifecycle.go`, cuyo `OnEnter` adopta la nueva firma y retorna `nil`; PR-05B1a1 MUST NOT propagar todavía errores físicos reales de desbloqueo.

#### Scenario: Error de `OnEnter` con destino committed

- GIVEN una transición FSM válida desde `LOCKED` hacia `UNLOCKED`
- AND un `OnEnter` que retorna un error identificable
- WHEN se envía el evento
- THEN `Send` MUST retornar ese error preservando `errors.Is`
- AND el callback MUST haber recibido `from=LOCKED` y `to=UNLOCKED` exactamente una vez
- AND el destino `UNLOCKED` MUST permanecer committed sin rollback automático

#### Scenario: Transición inválida

- GIVEN un evento que no es válido desde el estado actual
- WHEN se envía el evento
- THEN la FSM MUST rechazarlo
- AND MUST NOT invocar `OnEnter`

### Requirement: Observabilidad de postcondición de ventana (PR-05B1a2a)

`WindowManager.SetWidgetMode` MUST eliminar fullscreen y confirmar la postcondición con `runtime.WindowIsFullscreen` antes de actualizar bookkeeping interno. Fullscreen aún activo, o un estado crítico que no pueda verificarse y resulte `UNKNOWN`, MUST ser FAILURE; bookkeeping interno MUST NOT sustituir la verificación runtime coherente. Always-on-top, resize y center son operaciones cosméticas best-effort y MUST NOT hacer fallar automáticamente el desbloqueo. PR-05B1a2a MUST NOT cambiar `KeyboardHook`, `unlockCore`, watchdog, `SessionActivationCoordinator` ni persistencia de estado de sesión.

#### Scenario: Postcondición crítica de ventana

- GIVEN que se solicitó salir de fullscreen
- WHEN runtime confirma fullscreen falso
- THEN la postcondición crítica MUST ser SUCCESS
- BUT WHEN runtime confirma fullscreen verdadero o no puede verificar el estado crítico
- THEN el resultado MUST ser FAILURE

### Requirement: Observabilidad de hook, `unlockCore` y watchdog (PR-05B1a2b)

`KeyboardHookWindows.Remove` MUST propagar el resultado de `UnhookWindowsHookEx`, conservar estado active/handle veraz ante fallo y limpiarlo solamente en éxito. `PostThreadMessageW` es cleanup best-effort; un fallo suyo MUST NOT ocultar el resultado crítico ni falsear `threadID`. `unlockCore()` MUST devolver error, agregar errores críticos con `errors.Join` y continuar el cleanup aunque una operación aislada falle. `app_lifecycle` MUST propagar el error físico. El watchdog MUST eliminar la remoción duplicada del hook, ejecutar exactamente una ruta física `EventUnlock`, exponer el fallo, no reintentar y no originar `SessionActivation`. Esta unidad excluye `SessionActivationCoordinator`, persistencia de estado de sesión, `station_login_enabled`, gates `App.Unlock`/Dispatcher y recuperación PR-06; solo puede incluir la regresión de mantenimiento requerida por la propagación física de `OnEnter`.

SUCCESS físico MUST requerir ambas postcondiciones críticas confirmadas: fullscreen removido y keyboard hook removido. Un resultado crítico `UNKNOWN` MUST ser FAILURE. Las operaciones cosméticas/best-effort MUST NOT hacer fallar automáticamente el desbloqueo. La FSM MAY haber transitado `LOCKED → UNLOCKED` antes del error físico: el error MUST ser observable y retornado, el destino MUST permanecer committed y el sistema MUST NOT ejecutar rollback automático. PR-06 es dueño exclusivo de recuperación o reconciliación posterior.

#### Scenario: Fallos aislados y cleanup continuo

- GIVEN que falla la verificación de ventana, la remoción del hook o ambas
- WHEN `unlockCore` procesa el desbloqueo
- THEN MUST devolver el fallo aislado o los fallos unidos
- AND MUST continuar el cleanup aplicable
- AND MUST NOT informar SUCCESS si falta alguna confirmación crítica

#### Scenario: Watchdog observable sin ruta de sesión

- GIVEN un desbloqueo iniciado por WATCHDOG
- WHEN ocurre un fallo físico
- THEN MUST existir exactamente una ruta física `EventUnlock` y ninguna remoción duplicada del hook
- AND MUST exponer el error sin retry ni `SessionActivation`
- AND MUST conservar la naturaleza privilegiada `RECOVERY`

### Requirement: Barrera normal, estado, repetición y conflicto (PR-05B1b posterior)

PR-05B1b MUST hacer a `SessionActivationCoordinator` la única ruta normal que origine `EventUnlock`, solo después del commit durable de `PersistSessionActivation`. Debe consumir el resultado físico calificado de PR-05B1a2b: tras éxito, MUST intentar `pending_unlock` → `active`; tras fallo, MUST conservar Session, motivos, `SyncIntent` y claim, MUST intentar `unlock_failed` y MUST devolver fallo. SQLite y el desbloqueo físico MUST NOT suponerse atómicos. Si falla una actualización posterior, el estado durable MAY permanecer `pending_unlock` y el resultado inmediato MUST ser ambiguo/requiere recuperación, nunca éxito, estación libre ni sesión activa. No debe borrar, compensar, liberar claim ni reintentar automáticamente. PR-06 es dueño de recuperación tras reinicio, reconstrucción al arranque, temporizador durable y tratamiento de estos estados.

#### Scenario: Resultado físico calificado y actualización de estado

- GIVEN una sesión durable en `pending_unlock`
- WHEN B1b recibe SUCCESS calificado de PR-05B1a2b
- THEN MUST intentar persistir `Session.status = active`
- AND MUST informar fallo o ambigüedad si esa actualización no se confirma

#### Scenario: Fallo físico calificado

- GIVEN una sesión durable en `pending_unlock`
- WHEN B1b recibe FAILURE de PR-05B1a2b
- THEN MUST intentar persistir `Session.status = unlock_failed`
- AND MUST devolver fallo sin éxito simulado ni compensación

### Requirement: Repetición y conflicto de activación (PR-05B1b posterior)

PR-05B1b, no PR-05B1a1/PR-05B1a2a/PR-05B1a2b, MUST consultar el agregado durable por `session_id` antes de crear o desbloquear. Para el mismo `session_id` y agregado idéntico, si el estado es `active` MAY devolver éxito idempotente sin crear Session, claim ni `SyncIntent`, ni desbloquear de nuevo. Si el estado es `pending_unlock` o `unlock_failed`, MUST devolver el estado existente para recuperación y MUST NOT devolver éxito automático. El mismo `session_id` con datos distintos MUST resultar en `CONFLICT` sin mutación. Un claim de estación o de `(center_id,user_id)` perteneciente a otra sesión MUST resultar en `CONFLICT` y conservarse. Ante timeout o resultado desconocido, el llamador MUST reutilizar el mismo `session_id` y consultar el estado durable; MUST NOT asignar un ID nuevo solo por timeout, y la respuesta MUST distinguir no persistido, `pending_unlock`, `unlock_failed`, `active` y `CONFLICT`.

#### Scenario: Repetición idéntica de una sesión activa

- GIVEN un agregado durable idéntico con el mismo `session_id` y estado `active`
- WHEN llega una repetición
- THEN el sistema MAY devolver éxito idempotente
- AND MUST NOT crear otra sesión, claim o `SyncIntent`
- AND MUST NOT volver a desbloquear la estación

#### Scenario: Repetición pendiente, fallida o conflictiva

- GIVEN una repetición con el mismo `session_id`
- WHEN el agregado durable está `pending_unlock` o `unlock_failed`, contiene datos distintos, o el recurso está reclamado por otra sesión
- THEN el sistema MUST devolver respectivamente el estado existente o `CONFLICT`
- AND MUST NOT mutar el agregado ni crear otro

### Requirement: Compatibilidad y excepciones privilegiadas de desbloqueo (PR-05B2 posterior)

PR-05B2 MUST clasificar `App.Unlock` como REFACTOR: MUST NOT conservar acceso directo normal a `EventUnlock` ni originar `SessionActivation`. Puede permanecer temporalmente como superficie de compatibilidad hasta verificar llamadores externos, pero MUST NOT sortear la barrera normal ni usar datos sintéticos; su eliminación no queda autorizada todavía. Con `station_login_enabled=true`, un `UNLOCK` legado sin payload MUST NOT crear una sesión nueva ni ser excepción a commit-antes-de-unlock: el inicio normal exige un agregado válido y MUST NOT fabricar usuario, sesión, centro, motivo ni `SessionActivation`. La compatibilidad feature-OFF se preserva sin convertir el flujo legado en bypass; el contrato LAN estructurado exacto queda para PR-07.

RECOVERY/WATCHDOG y MAINTENANCE/ADMIN son excepciones privilegiadas al flujo de usuario. PR-05B1a2b preserva su naturaleza al propagar resultados físicos veraces: RECOVERY/WATCHDOG MUST seguir una ruta explícita, identificable y auditable como `RECOVERY` con causa técnica, sin datos sintéticos ni `SessionActivation`, y puede beneficiarse de esa observabilidad sin convertirse en activación de sesión. MAINTENANCE/ADMIN MUST seguir una ruta explícita, identificable y auditable con identidad de mantenimiento, que preserve `Maintenance authentication → MaintenanceUnlock → EventMaintenanceUnlock` y sus garantías existentes de autenticación/autorización. PR-05B2 cubre sus regresiones, sin convertir ninguna excepción en bypass normal.

#### Scenario: UNLOCK legado sin payload con login habilitado

- GIVEN `station_login_enabled=true`
- AND un comando legado `UNLOCK` sin payload
- WHEN se procesa el comando
- THEN MUST NOT crear una sesión ni datos sintéticos
- AND MUST NOT activar la estación por la ruta normal

#### Scenario: Ruta privilegiada separada

- GIVEN una causa técnica de RECOVERY/WATCHDOG o una identidad MAINTENANCE/ADMIN autorizada
- WHEN se solicita el desbloqueo privilegiado aplicable
- THEN el sistema MUST usar su ruta explícita y auditable, sin `SessionActivation` sintética
- AND MUST mantenerla separada de `SessionActivationCoordinator` y del inicio normal de usuario

### Requirement: Discriminación sin downgrade y frontera v2 inerte (PR-07)

PR-07 MUST tratar un objeto como candidato v2 cuando esté presente `protocol_version` o cualquiera de estos marcadores exclusivos: `message_id`, `kind`, `correlation_id`, `idempotency_key`, `source_id`, `station_id`, `center_id`, `link_id`, `connection_epoch`, `sequence`, `sent_at` o `payload_schema_version`. Un candidato v2 MUST pasar solamente al decoder v2 y MUST NOT usar fallback legado por campo faltante, tipo incorrecto, versión/schema no soportado ni otra invalidez estructural. Solo un frame sin marcadores v2 MAY usar el decoder legado vigente.

Como el WebSocket de PR-07 no está autenticado, todo frame v2 MUST permanecer sin autoridad: el sistema MAY decodificarlo, validarlo, clasificarlo, deduplicarlo por `message_id` dentro de la conexión y entregarlo a un callback inerte/unsupported, pero MUST NOT enviarlo al Dispatcher legado ni permitir que produzca desbloqueo, bloqueo privilegiado, temporizador, apagado, mantenimiento, activación/coordinación de sesión o persistencia. Un error de parse v2 MUST NOT terminar el proceso, provocar reconexión por sí solo ni impedir que el read loop continúe; el sistema MUST NOT registrar el payload v2 completo.

#### Scenario: Marcador v2 sin versión no degrada a legado

- GIVEN un objeto `{"type":"UNLOCK","message_id":"m-1"}` que contiene un marcador exclusivo v2 pero omite `protocol_version`
- WHEN el sistema discrimina y decodifica el frame
- THEN MUST clasificarlo como v2 malformado
- AND MUST NOT intentar el decoder legado
- AND MUST NOT desbloquear ni producir otro efecto privilegiado

#### Scenario: Versión o schema v2 no soportado

- GIVEN un candidato v2 con otro entero en `protocol_version` o con un `payload_schema_version` positivo distinto de `1`
- WHEN el sistema valida el sobre
- THEN MUST rechazarlo como versión de protocolo o schema no soportado, según corresponda
- AND MUST NOT degradarlo a legado ni producir efectos privilegiados
- AND MUST continuar procesando frames posteriores de la conexión

#### Scenario: Tipo v2 desconocido pero estructuralmente válido

- GIVEN un sobre v2 estructuralmente válido cuyo `type` no está registrado
- WHEN el sistema lo clasifica y enruta en PR-07
- THEN MAY clasificarlo como v2 válido con routing unsupported
- AND MUST mantener el routing inerte
- AND MUST NOT reutilizar el Dispatcher legado ni producir efectos de producto

#### Scenario: Frame sin marcadores conserva compatibilidad legada

- GIVEN un objeto legado `{ "type": "...", "payload": ... }` sin ningún marcador v2
- WHEN el sistema discrimina el frame
- THEN MAY usar el decoder legado vigente
- AND MUST preservar su manejo actual de comandos conocidos y su rechazo actual de comandos desconocidos

### Requirement: Separación de credenciales y autorizaciones de PIN

El sistema MUST modelar como conceptos y contratos separados: (1) la credencial permanente, compuesta por un verificador o hash no reversible del PIN personal; (2) la credencial temporal de cuatro dígitos emitida para `EMAIL_RECOVERY`, con ciclo de vida y validación independientes; y (3) la autorización o grant de restablecimiento, que concede temporalmente una operación de cambio de PIN pero no es un PIN ni una credencial de login. El sistema MUST NOT reutilizar, convertir ni interpretar uno de estos conceptos como cualquiera de los otros.

#### Scenario: Validación de la credencial permanente

- GIVEN que un usuario tiene un PIN personal vigente
- WHEN el sistema conserva o valida su credencial permanente
- THEN el sistema MUST usar solamente un verificador o hash no reversible apropiado para validación
- AND MUST NOT poder reconstruir el PIN personal desde esa representación

#### Scenario: Credencial temporal independiente

- GIVEN que se genera un PIN temporal para `EMAIL_RECOVERY`
- WHEN el sistema registra su vigencia y estado
- THEN MUST tratarlo como una credencial temporal independiente de la credencial permanente
- AND MUST NOT sustituir el verificador permanente por el PIN temporal

#### Scenario: Grant de restablecimiento no reutilizable como credencial

- GIVEN una autorización vigente para restablecer un PIN
- WHEN se presenta fuera de la operación, solicitud, usuario o estación a los que esté vinculada
- THEN el sistema MUST rechazarla
- AND MUST NOT aceptarla como PIN permanente, PIN temporal de correo ni credencial de login

### Requirement: Protección del PIN y bloqueo por intentos fallidos

El sistema MUST proteger el PIN personal mediante un verificador o hash no reversible con salt, o una protección equivalente, y MUST NOT almacenarlo en texto plano. El PIN actual, el PIN nuevo, el PIN temporal de correo, cualquier verificador o hash, la autorización de restablecimiento y sus secretos asociados MUST NOT aparecer en logs, auditorías, telemetría, mensajes técnicos o diagnósticos, errores ni trazas. Los verificadores, hashes, autorizaciones o grants y secretos asociados MUST NOT aparecer en interfaces administrativas; ningún PIN MUST aparecer allí salvo en una superficie transitoria de captura que sea estrictamente necesaria y haya sido aprobada para el actor correspondiente. El PIN temporal MUST exponerse únicamente al usuario mediante el correo de recuperación destinado a su dirección registrada y mediante la superficie transitoria destinada a introducirlo. El sistema MUST limitar intentos fallidos y aplicar bloqueo conforme a valores configurables; dichos valores MUST NOT estar codificados de forma fija.

#### Scenario: Persistencia de una credencial permanente

- GIVEN que un usuario crea o cambia su PIN personal de cuatro dígitos
- WHEN el sistema conserva la credencial permanente
- THEN MUST persistir solamente una representación no reversible apropiada para validación
- AND MUST NOT persistir ni emitir el PIN personal en texto plano

#### Scenario: Manejo seguro de secretos de recuperación

- GIVEN que un flujo de recuperación genera o consume un PIN temporal o una autorización de restablecimiento
- WHEN el sistema registra actividad, informa un error, produce telemetría o presenta información administrativa
- THEN MUST omitir el PIN temporal, la autorización y cualquier secreto asociado
- AND MUST limitar la exposición del PIN temporal al correo registrado y a su introducción transitoria por el usuario

#### Scenario: Se alcanza el límite configurado

- GIVEN una política configurada de intentos máximos y duración de bloqueo
- WHEN los intentos fallidos alcanzan el límite aplicable
- THEN el sistema MUST bloquear nuevos intentos conforme a esa política
- AND MUST auditar el bloqueo sin registrar ningún PIN, verificador, hash, autorización ni secreto asociado

### Requirement: Modos diferenciados de recuperación y restablecimiento de PIN

El sistema MUST distinguir y registrar los modos estables `EMAIL_RECOVERY`, `ASSISTED_STATION_RESET` y `DIRECT_DINAMIZADOR_RESET`. Cada modo MUST conservar su propio origen, actores, condiciones de autorización y ciclo de vida, y MUST NOT presentarse ni procesarse como si fuera cualquiera de los otros modos.

#### Scenario: Selección explícita del modo

- GIVEN que se inicia una recuperación o un restablecimiento de PIN
- WHEN el sistema crea la solicitud correspondiente
- THEN MUST asignarle exactamente uno de los modos `EMAIL_RECOVERY`, `ASSISTED_STATION_RESET` o `DIRECT_DINAMIZADOR_RESET`
- AND MUST conservar ese modo durante todo el ciclo de vida de la solicitud

### Requirement: Recuperación por correo `EMAIL_RECOVERY`

Cuando el usuario solicite `EMAIL_RECOVERY`, el sistema MUST comprobar que dispone de un correo registrado y que el canal central correspondiente está disponible. La plataforma responsable del canal MUST generar un PIN temporal independiente de exactamente cuatro dígitos y MUST enviarlo solamente al correo registrado; MUST NOT recuperar, revelar ni enviar el PIN personal permanente actual. El PIN temporal MUST tener una vigencia limitada por política, MUST expirar, MUST permitir una sola finalización satisfactoria y MUST quedar invalidado después de esa finalización. Si debe conservarse para validación, MUST almacenarse solamente mediante una representación protegida y MUST NOT persistirse permanentemente en texto plano. La emisión de un nuevo PIN temporal MUST invalidar cualquier PIN temporal activo anterior para el mismo usuario y propósito. Después de validar satisfactoriamente el PIN temporal, el flujo MUST exigir al usuario establecer y confirmar un nuevo PIN personal antes de completar el restablecimiento, y MUST NOT convertir el PIN temporal en credencial permanente. La implementación del servicio central de entrega de correo queda fuera del alcance de este cambio, pero la futura plataforma central MUST cumplir este contrato.

#### Scenario: Emisión de PIN temporal por correo

- GIVEN un usuario con correo registrado
- AND están disponibles Internet, la plataforma correspondiente y el canal de correo
- WHEN el usuario solicita `EMAIL_RECOVERY`
- THEN la plataforma responsable MUST generar un PIN temporal independiente de cuatro dígitos
- AND MUST enviarlo solamente al correo registrado
- AND MUST conservar para validación únicamente una representación protegida cuando sea necesario
- AND MUST NOT persistir permanentemente el PIN temporal en texto plano
- AND MUST NOT enviar el PIN personal permanente actual

#### Scenario: Recuperación por correo no disponible

- GIVEN que falta correo registrado, Internet, plataforma correspondiente o canal de correo
- WHEN el usuario solicita `EMAIL_RECOVERY`
- THEN el sistema MUST informar que ese modo no está disponible sin revelar datos sensibles
- AND MUST NOT generar una recuperación por correo utilizable
- AND MUST permitir que el usuario acuda a uno de los modos de restablecimiento con Dinamizador

#### Scenario: Un nuevo PIN temporal reemplaza al anterior

- GIVEN que existe un PIN temporal activo para `EMAIL_RECOVERY`
- WHEN la plataforma emite un nuevo PIN temporal para el mismo usuario y propósito
- THEN el sistema MUST invalidar el PIN temporal anterior
- AND MUST aceptar solamente el nuevo PIN temporal mientras permanezca vigente

#### Scenario: PIN temporal usado satisfactoriamente

- GIVEN un PIN temporal vigente de `EMAIL_RECOVERY`
- WHEN el usuario lo valida satisfactoriamente y establece y confirma un nuevo PIN personal
- THEN el sistema MUST completar el restablecimiento con la nueva credencial permanente protegida
- AND MUST invalidar el PIN temporal para impedir otro uso
- AND MUST invalidar la credencial permanente anterior solamente después de confirmar satisfactoriamente la nueva

#### Scenario: PIN temporal expirado o consumido

- GIVEN un PIN temporal expirado, ya consumido o invalidado por reemplazo
- WHEN se intenta utilizarlo
- THEN el sistema MUST rechazarlo
- AND MUST NOT modificar la credencial permanente vigente

### Requirement: Restablecimiento asistido en estación `ASSISTED_STATION_RESET`

En `ASSISTED_STATION_RESET`, el usuario MUST iniciar la solicitud en Usuario PC y el sistema MUST remitirla al Dinamizador. La solicitud MUST identificar internamente al usuario, la solicitud, la estación y la Infoplaza, además de incluir el documento solamente cuando sea necesario para la identificación operativa presencial; MUST NOT incluir ningún PIN, verificador, hash, grant ni secreto. El Dinamizador MUST validar presencialmente la identidad y MUST aprobar o rechazar la solicitud. Una aprobación MUST habilitar temporalmente el restablecimiento para ese usuario, solicitud y estación mediante una autorización de un solo uso, con vigencia limitada y resistente a repetición. El usuario MUST ingresar y confirmar personalmente el nuevo PIN en la estación, y el Dinamizador MUST NOT conocer el PIN anterior ni el nuevo. La autorización MUST invalidarse por éxito, rechazo, expiración, cancelación o reemplazo.

#### Scenario: La estación envía una solicitud asistida

- GIVEN que el usuario solicita restablecer su PIN desde una estación
- WHEN Usuario PC remite la solicitud `ASSISTED_STATION_RESET` al Dinamizador
- THEN la solicitud MUST incluir los identificadores internos de usuario, solicitud, estación e Infoplaza, la fecha y hora y el tipo de solicitud
- AND MAY incluir el documento únicamente para la identificación operativa presencial
- AND MUST NOT incluir ningún PIN, verificador, hash, grant ni secreto asociado

#### Scenario: Solicitud asistida aprobada

- GIVEN que el usuario inició una solicitud `ASSISTED_STATION_RESET`
- AND el Dinamizador validó presencialmente su identidad
- WHEN el Dinamizador aprueba la solicitud vigente
- THEN el sistema MUST habilitar una autorización temporal de un solo uso vinculada al usuario, la solicitud y la estación
- AND MUST exigir que el usuario ingrese y confirme personalmente el nuevo PIN en esa estación
- AND MUST impedir que el Dinamizador vea el PIN anterior o el nuevo

#### Scenario: Consumo satisfactorio de autorización asistida

- GIVEN una autorización vigente de `ASSISTED_STATION_RESET`
- WHEN el usuario confirma satisfactoriamente el nuevo PIN en la estación vinculada
- THEN el sistema MUST consumir e invalidar la autorización
- AND MUST activar la nueva credencial permanente protegida
- AND MUST invalidar la credencial permanente anterior solamente después de esa confirmación satisfactoria

#### Scenario: Repetición o ámbito incorrecto de autorización asistida

- GIVEN una autorización de `ASSISTED_STATION_RESET` ya consumida o vinculada a otro usuario, solicitud o estación
- WHEN se intenta repetirla o utilizarla fuera de su ámbito
- THEN el sistema MUST rechazarla
- AND MUST NOT cambiar la credencial permanente vigente

#### Scenario: Solicitud asistida rechazada, expirada, cancelada o reemplazada

- GIVEN una solicitud o autorización de `ASSISTED_STATION_RESET`
- WHEN es rechazada, expira, se cancela o es reemplazada por una nueva solicitud para el mismo propósito
- THEN la autorización anterior MUST quedar inválida
- AND MUST NOT permitir cambiar el PIN con esa autorización

### Requirement: Restablecimiento administrativo directo `DIRECT_DINAMIZADOR_RESET`

En `DIRECT_DINAMIZADOR_RESET`, un Dinamizador autorizado MUST validar presencialmente la identidad del usuario e iniciar el restablecimiento administrativo desde su software. El sistema MUST NOT exigir el PIN anterior y MUST NOT mostrarlo. El flujo administrativo autorizado MUST permitir establecer y confirmar una nueva credencial permanente protegida; la credencial anterior MUST continuar vigente hasta que la nueva quede confirmada satisfactoriamente y MUST invalidarse solamente después de esa confirmación. La operación MUST permanecer diferenciada de `ASSISTED_STATION_RESET`: MUST NOT exigir una solicitud originada por Usuario PC ni reutilizar silenciosamente el flujo o la autorización vinculada a estación de ese modo. La persona que introduce el nuevo PIN y la superficie exacta de captura permanecen como OPEN REQUIREMENT; hasta resolverlo, el sistema MUST preservar estas semánticas administrativas sin asumir que la captura ocurre en Usuario PC.

#### Scenario: Inicio administrativo directo

- GIVEN que un Dinamizador autorizado validó presencialmente la identidad del usuario
- WHEN inicia `DIRECT_DINAMIZADOR_RESET` desde su software
- THEN el sistema MUST autorizar el flujo administrativo sin solicitar el PIN anterior
- AND MUST NOT mostrar el PIN anterior
- AND MUST identificar la operación como `DIRECT_DINAMIZADOR_RESET`

#### Scenario: Confirmación satisfactoria del restablecimiento directo

- GIVEN un flujo `DIRECT_DINAMIZADOR_RESET` autorizado y vigente
- WHEN la nueva credencial permanente queda establecida y confirmada satisfactoriamente mediante la superficie que se apruebe
- THEN el sistema MUST activar la nueva credencial protegida
- AND MUST invalidar la credencial anterior solamente después de esa confirmación
- AND MUST consumir o cerrar la autorización administrativa para impedir repetición

#### Scenario: Restablecimiento directo incompleto

- GIVEN un flujo `DIRECT_DINAMIZADOR_RESET` autorizado
- WHEN la nueva credencial no llega a confirmarse o la operación expira, se cancela o falla
- THEN el sistema MUST conservar vigente la credencial permanente anterior
- AND MUST impedir que la autorización incompleta se reutilice después de su estado terminal

### Requirement: Visualización pública y minimización de datos

Las pantallas públicas de Usuario PC MUST mostrar únicamente la información necesaria para el uso de la sesión. Durante una sesión, el sistema MAY mostrar avatar, primer nombre, inicial del apellido, tiempo y costo cuando aplique. MUST NOT exponer documento completo, correo, dirección, fecha de nacimiento, datos de discapacidad ni datos administrativos o privados innecesarios.

#### Scenario: Sesión visible en una estación pública

- GIVEN una sesión activa
- WHEN Usuario PC presenta la identidad y el estado de la sesión
- THEN MUST limitar la visualización al avatar, primer nombre, inicial del apellido, tiempo y costo aplicable
- AND MUST NOT mostrar datos personales o administrativos adicionales

### Requirement: Auditoría segura y trazable

El sistema MUST auditar, cuando correspondan al componente y al flujo, activación de estación, actualización de software, registro de usuario, cambios de indicador de discapacidad sin valores de tipo, autenticación, intentos fallidos, inicio y fin de sesión, solicitudes de tiempo, aprobaciones y rechazos, modificaciones de tiempo, transferencias, bloqueos, solicitudes de recuperación de PIN, aprobación o rechazo de recuperación, restablecimientos de PIN, envío de correo de recuperación, cambios de configuración, errores relevantes y eventos de sincronización o conciliación. Para una recuperación o restablecimiento, la auditoría MUST incluir los identificadores internos del solicitante, actor y autorizador cuando existan; el modo `EMAIL_RECOVERY`, `ASSISTED_STATION_RESET` o `DIRECT_DINAMIZADOR_RESET`; el usuario afectado; la estación cuando sea relevante; la Infoplaza; las fechas y horas aplicables; el resultado; y los hechos de expiración, cancelación o consumo cuando correspondan. Para los demás eventos, MUST incluir fecha y hora, resultado y, cuando aplique, identificadores internos de usuario, estación, sesión, operador e Infoplaza. La auditoría MUST preferir identificadores internos sobre la duplicación de datos personales y MUST NOT incluir el PIN actual, nuevo o temporal, verificadores, hashes, autorizaciones o grants de restablecimiento, contraseñas, tokens, secretos, archivos privados completos ni contenido innecesario de mensajes.

#### Scenario: Evento operativo auditable

- GIVEN que ocurre un inicio de sesión, una aprobación, un rechazo, una transferencia o una conciliación
- WHEN se registra el evento
- THEN la auditoría MUST identificar el tipo, fecha y hora, resultado y entidades internas aplicables
- AND MUST permitir relacionarlo con la estación y la sesión cuando existan

#### Scenario: Evento de recuperación o restablecimiento auditable

- GIVEN que se solicita, autoriza, rechaza, completa, cancela, reemplaza o expira una recuperación o restablecimiento de PIN
- WHEN se registra el evento
- THEN la auditoría MUST identificar solicitante, actor y autorizador cuando existan, modo estable, usuario afectado, estación cuando sea relevante, Infoplaza, fechas y horas, resultado y estado de expiración, cancelación o consumo aplicable
- AND MUST NOT incluir ningún PIN, verificador, hash, autorización, grant, token, secreto ni contenido innecesario del mensaje

#### Scenario: Diagnóstico relacionado con credenciales

- GIVEN que un intento fallido, bloqueo, recuperación o restablecimiento produce un log, error, traza, telemetría o mensaje técnico
- WHEN el sistema registra o presenta el diagnóstico
- THEN MUST limitarlo a códigos de resultado e identificadores mínimos no secretos
- AND MUST NOT exponer ningún PIN, verificador, hash, autorización, grant ni secreto asociado

### Requirement: Autoridad y conciliación

Durante una desconexión, Usuario PC MUST registrar como hechos locales sus autenticaciones, sesiones, transferencias y cambios de estado ocurridos legítimamente. El Dinamizador MUST actuar como autoridad operativa de la Infoplaza para autorizar y conciliar sesiones autónomas, transferencias y estados locales. La plataforma central MUST ser la autoridad nacional para usuarios, catálogos, conflictos entre centros y consolidación. Al recuperarse la comunicación, los hechos pendientes MUST enviarse para conciliación sin duplicarlos ni perder su procedencia, modalidad o trazabilidad; una discrepancia MUST conservarse como pendiente o conflicto explícito hasta que la autoridad aplicable la resuelva. Como invariante futuro acotado, los duplicados offline MUST NOT convertirse silenciosamente en identidades independientes: los hechos locales se registran y posteriormente se concilian o se marcan conflictos cuando corresponda. Este cambio no diseña el comportamiento de conciliación.

#### Scenario: Conciliación de una sesión autónoma

- GIVEN una sesión autónoma registrada localmente durante una interrupción
- WHEN se recupera la comunicación con el Dinamizador
- THEN Usuario PC MUST presentar el hecho local para conciliación
- AND el Dinamizador MUST reconocerlo, conciliarlo o marcar una discrepancia explícita
- AND el sistema MUST NOT crear una sesión duplicada

#### Scenario: Conflicto entre Infoplazas

- GIVEN que la sincronización detecta estados de usuario o sesiones incompatibles entre centros
- WHEN el conflicto requiere autoridad nacional
- THEN el sistema MUST conservar la trazabilidad de los hechos involucrados
- AND MUST remitir el conflicto a la autoridad central sin resolverlo mediante sobrescritura local silenciosa

### Requirement: Compatibilidad transitoria con el flujo dirigido por Dinamizador (PR-05B2 posterior)

PR-05B2 MUST preservar la compatibilidad con la función desactivada y sus barreras aprobadas. El nuevo login originado en Usuario PC MUST convivir durante la transición con el flujo existente dirigido por el Dinamizador. Ambos flujos MUST producir una interpretación única y coherente de usuario, estación, sesión, tiempo, modalidad y estado; MUST respetar las mismas reglas de unicidad, durabilidad y auditoría. La compatibilidad normal MUST respetar la barrera de `SessionActivationCoordinator` y MUST NOT usar `App.Unlock` directo, `UNLOCK` sin payload ni datos sintéticos para crear una sesión. Cuando el nuevo flujo no esté habilitado o no sea compatible con los participantes desplegados, el sistema MUST conservar el flujo histórico dirigido por el Dinamizador como contención, sin borrar sesiones ni hechos pendientes.

#### Scenario: Convivencia durante despliegue gradual

- GIVEN una Infoplaza con componentes en transición
- WHEN una sesión se inicia desde Usuario PC o desde el flujo dirigido por Dinamizador
- THEN el sistema MUST aplicar las mismas reglas de sesión activa, persistencia y auditoría
- AND MUST evitar modelos de sesión paralelos o contradictorios

#### Scenario: Nuevo login deshabilitado

- GIVEN que el nuevo login se encuentra deshabilitado para una estación o Infoplaza
- WHEN se requiere iniciar una sesión
- THEN el flujo dirigido por el Dinamizador MUST permanecer disponible
- AND la desactivación MUST NOT eliminar sesiones, auditoría ni transacciones pendientes existentes

### Requirement: Vínculo seguro de Usuario PC conforme a PR-08

Usuario PC MUST actuar como cliente WSS sobre TLS 1.3 con mTLS para todo tráfico LAN que aspire a elegibilidad privilegiada. MUST usar una identidad/certificado único por instalación emitido por la CA privada controlada por Infoplazas, verificar la identidad de instalación y centro del Dinamizador y tratar `station_id`, `center_id`, `source_id` y `link_id` como claims hasta contrastarlos con el certificado y registro autorizados. WS en claro, contraseñas compartidas y handshakes criptográficos propios MUST NOT conferir confianza.

La clave privada MUST generarse y permanecer en la instalación bajo almacenamiento Windows protegido y ACL restrictiva. El spike de almacenamiento MUST elegir Certificate Store/CNG o un adaptador DPAPI que demuestre ausencia de ruta de texto plano, exportación o copia y protección equivalente. Usuario PC MUST poder autenticarse runtime sin Internet con credenciales locales no expiradas, trust root público fijado y estado local conocido.

#### Scenario: Usuario PC verifica al Dinamizador offline

- GIVEN Usuario PC no tiene Internet
- AND conserva una credencial local vigente, trust root fijado y un registro autorizado del Dinamizador para su centro
- WHEN establece TLS 1.3 mTLS y WSS
- THEN MUST verificar certificado, identidad de instalación, autorización y binding de centro
- AND MAY continuar a negociación sin ejecutar acción de producto

#### Scenario: Identidad o centro del Dinamizador no coincide

- GIVEN mTLS completa criptográficamente
- BUT el certificado, registro o claim no corresponde al Dinamizador y centro autorizados
- WHEN Usuario PC evalúa el peer
- THEN MUST impedir `ACTIVE`
- AND MUST cerrar y auditar de forma fail-closed sin confiar en el claim discrepante

### Requirement: Negociación, reconexión y anti-replay de Usuario PC

Usuario PC MUST seguir la FSM `DISCONNECTED → CONNECTED_UNAUTHENTICATED → AUTHENTICATING → AUTHENTICATED_NEGOTIATING → ACTIVE`, con rutas `REJECTED/CLOSING`. Cada conexión nueva MUST comenzar sin auth, época, secuencia, capacidades ni privilegios heredados. TLS 1.3 early data/0-RTT MUST estar deshabilitado; resumption solo MAY usarse con revalidación completa por conexión.

Después de mTLS, Usuario PC MUST emitir `link.hello` sobre Agent v2 protocolo `2`, schema `1`, anunciando `secure_link_v1`; MUST aceptar únicamente `link.accept` correlacionado, con claims vinculados, intersección allowlisted que incluya esa capacidad y `connection_epoch` aleatorio no cero asignado por Dinamizador. Los frames de handshake MUST carecer de secuencia. Todo frame v2 post-ACTIVE MUST usar la época actual y secuencia direccional `uint64` desde `1` con incremento exacto `+1`; duplicate/lower MUST ser replay, gap MUST auditarse y cerrar, y overflow MUST cerrar antes de wrap. `sent_at` MUST ser observacional, no defensa primaria.

#### Scenario: Accept válido activa el vínculo

- GIVEN Usuario PC envió un `link.hello` válido después de mTLS
- WHEN recibe un `link.accept` con correlación exacta, binding válido, época no cero e intersección allowlisted que incluye `secure_link_v1`
- THEN MUST entrar en `ACTIVE`
- AND MUST habilitar únicamente elegibilidad técnica, no acciones de producto

#### Scenario: Reconexión o replay de época anterior

- GIVEN un vínculo anterior tuvo una época y secuencias activas
- WHEN Usuario PC reconecta o recibe tráfico con la época anterior
- THEN MUST reiniciar autenticación y negociación desde cero
- AND MUST rechazar la época anterior como stale/replay

### Requirement: Rechazo, auditoría y ausencia de downgrade

Un fallo TLS previo al tráfico de aplicación MUST producir auditoría local de seguridad y no `link.reject`; un TLS alert MUST NOT interpretarse como frame JSON. Usuario PC MAY procesar `link.reject` únicamente después de mTLS y con correlación segura al hello. MUST conservar la allowlist de categorías aprobada en la especificación cross-component.

Usuario PC MUST usar un `SecurityAuditSink` local durable con allowlist minimizada. Si la auditoría obligatoria no está disponible, la elegibilidad privilegiada MUST permanecer falsa. El parser legado MAY permanecer, pero ninguna mutación privilegiada legada no autenticada MUST ejecutarse y un fallo v2 MUST NOT degradar a privilegios v1.

#### Scenario: TLS falla sin canal JSON

- GIVEN el certificado del Dinamizador es inválido, expirado, revocado localmente o no confiable
- WHEN falla TLS antes del tráfico de aplicación
- THEN Usuario PC MUST impedir la conexión privilegiada y auditar localmente
- AND MUST NOT esperar ni fabricar `link.reject`

#### Scenario: Falla el sink de auditoría obligatorio

- GIVEN mTLS y negociación válidos
- BUT el sink durable obligatorio no está disponible
- WHEN Usuario PC evalúa elegibilidad privilegiada
- THEN MUST conservarla falsa
- AND MAY mantener solo un vínculo técnicamente autenticado para operación no privilegiada

### Requirement: Alcance de PR-08U y cero acciones de producto

PR-08U MUST dividirse en PR-08U1 — credencial protegida, cliente WSS/mTLS y peer binding — y PR-08U2 — negociación, FSM, época/secuencia, reconexión y auditoría. Ambas unidades MUST implementar cero handlers de producto, migraciones o conducta de sesión. La autenticación MUST ser necesaria y no suficiente: futura elegibilidad requiere además identidad/centro, versión, capacidad, época, secuencia, allowlist de operación y autorización posterior de workflow. La conformidad cross-repo A+B+U MUST verificarse antes de PR-11.

#### Scenario: PR-08U termina sin activar producto

- GIVEN PR-08U1 y PR-08U2 conformes al contrato seguro
- WHEN el vínculo alcanza `ACTIVE`
- THEN MUST ejecutar cero `SessionActivation`, Coordinator, persistencia, `ACTIVATE_SESSION` o mutación de estación
- AND MUST esperar una entrega posterior autorizada para cualquier handler de producto

## OPEN REQUIREMENTS

Los nueve bloqueadores técnicos específicos de PR-08 están **CLOSED** en [`../../pr-08-blocking-requirements-closure.md`](../../pr-08-blocking-requirements-closure.md). Permanecen los requisitos no relacionados siguientes; ninguno se considera resuelto por el cierre de seguridad.

1. **OPEN REQUIREMENT — vigencia offline:** no está definido si la credencial local tendrá una antigüedad máxima ni, en su caso, el número exacto de días; una política futura MAY ser configurable y MUST NOT introducirse como constante fija sin aprobación.
2. **OPEN REQUIREMENT — intentos y bloqueo:** no están definidos el número exacto de intentos fallidos ni la duración exacta del bloqueo; ambos MUST ser configurables y no codificados de forma fija.
3. **OPEN REQUIREMENT — detección de interrupción:** no están definidos los tiempos exactos del heartbeat, reintentos, gracia o umbral para declarar no disponible al Dinamizador; MUST ser configurables o definidos en diseño sin alterar las condiciones funcionales de entrada al modo autónomo.
4. **OPEN REQUIREMENT — plataforma central:** los roles, permisos y reportes detallados de la futura plataforma web nacional que no sean necesarios para este cambio permanecen fuera de alcance y pendientes de definición.
5. **OPEN REQUIREMENT — captura en `DIRECT_DINAMIZADOR_RESET`:** falta aprobar quién introduce el nuevo PIN y en qué superficie se captura durante el restablecimiento administrativo directo. Esta decisión MUST NOT alterar que el Dinamizador inicia y autoriza el flujo desde su software, que no se exige ni muestra el PIN anterior, que la credencial anterior se invalida solo tras la confirmación satisfactoria de la nueva y que este modo no se confunde con `ASSISTED_STATION_RESET`.
6. **OPEN REQUIREMENT FUTURO NO BLOQUEANTE — sincronización de cambios de estado de sesión:** si la autoridad vigente requiere sincronizar transiciones `pending_unlock`, `active` o `unlock_failed`, debe aprobarse un contrato separado. Esta necesidad MUST NOT cambiar ni bloquear PR-05B1a1, PR-05B1a2a, PR-05B1a2b ni PR-05B1b, cuyo `SyncIntent` inicial conserva `aggregate_type=session`, versión `1` y estado `pending`.
