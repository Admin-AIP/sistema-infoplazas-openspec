# Propuesta: inicio de sesión/login de Usuario PC conforme a ADA Nova Plus

- **Cambio:** `session-start-login-usuario-pc`
- **Título:** Align Software Usuario PC session-start/login with ADA Nova Plus normative requirements
- **Estado:** Propuesta de producto; los nueve bloqueadores técnicos originales de PR-08 están **CLOSED** por arquitectura aprobada en [`pr-08-blocking-requirements-closure.md`](./pr-08-blocking-requirements-closure.md). Esto habilita planificación de implementación, no `apply`, creación de worktree ni cierre de los requisitos de negocio no relacionados.
- **Fuentes rectoras:** `docs/USUARIO PC - REQUERIMIENTOS PARA DESARROLLO.pdf` para Usuario PC; `docs/DINAMIZADOR - REQUERIMIENTOS PARA DESARROLLO.pdf` para las interacciones del Dinamizador; `docs/REQUERIMIENTOS PARA DESARROLLO.docx` para contexto general; guardrails, contexto del proyecto y decisiones de arquitectura técnica aprobadas para integración. Los PDFs exigen estaciones registradas/autorizadas, vínculo al centro, operación LAN offline, consideración de ciclo de vida/recuperación y auditoría; WSS/TLS 1.3 mTLS, negociación y anti-replay son decisiones de arquitectura técnica aprobadas, no requisitos criptográficos atribuidos directamente a los PDFs. Ante contradicción normativa, prevalece el documento aplicable y el conflicto debe resolverse antes de implementar.

## Intención y problema

La estación de Usuario PC hoy presenta una pantalla de bloqueo pasiva y depende principalmente de comandos enviados por el Dinamizador. No permite que un usuario registrado se autentique directamente con cédula/pasaporte y PIN, declare sus motivos de visita, seleccione actividades pertinentes ni inicie una sesión mediante todas las modalidades normativas. Tampoco dispone de persistencia local suficiente para autenticación, sesiones, catálogos, auditoría y sincronización offline-first.

Esta brecha obliga a utilizar un flujo manual concentrado en el Dinamizador, hace que la operación básica dependa de la comunicación disponible y dificulta explicar, auditar y sincronizar el origen real de cada sesión. La propuesta busca que el inicio de sesión desde la estación funcione de manera consistente, durable y trazable, sin eliminar las capacidades de control ya útiles ni depender permanentemente de Internet.

## Resultado de producto esperado

Un usuario previamente registrado y sincronizado podrá iniciar sesión desde una estación usando su cédula o pasaporte y un PIN de cuatro dígitos, registrar uno o más motivos de visita y, cuando corresponda, escoger una capacitación o actividad programada. Si no existen opciones programadas, el flujo continuará sin impedir el acceso.

Tras una autenticación y autorización válidas, la estación iniciará una sesión temporizada con la duración predeterminada configurable, registrará explícitamente su modalidad, conservará localmente la transacción para sincronización posterior, actualizará su estado operativo y mostrará un indicador de sesión no invasivo. El comportamiento cubrirá modalidad automática, modalidad autorizada por Dinamizador y modalidad autónoma ante una falla temporal de comunicación dentro de la política autorizada. Los eventos críticos serán auditables.

## Alcance propuesto

### Incluye

1. Convertir la pantalla de bloqueo de Usuario PC en un flujo de login con:
   - cédula o pasaporte como dato documental/de búsqueda;
   - PIN de exactamente cuatro dígitos;
   - selección obligatoria de uno o más motivos de visita;
   - listado condicional de capacitaciones/actividades programadas cuando se seleccione uno de esos motivos;
   - continuidad explícita del login cuando no existan actividades programadas.
2. Autenticar localmente a usuarios previamente sincronizados y elegibles, incluyendo operación sin Internet permanente.
3. Soportar y registrar las modalidades de inicio automática, autorizada por Dinamizador y autónoma por falla temporal de comunicación en modo autorizado.
4. Iniciar y conservar de forma durable una sesión temporizada con duración predeterminada configurable antes de considerar la estación habilitada.
5. Actualizar el estado visible de la estación y mostrar un widget de sesión no invasivo con la información mínima aprobada.
6. Registrar transacciones y eventos pendientes, reintentar su sincronización y evitar pérdida de datos durante desconexiones o reinicios.
7. Auditar como mínimo: login exitoso, login fallido, inicio de sesión, modalidad, pérdida/recuperación de comunicación y cambios/restablecimientos de PIN vinculados al flujo.
8. Cubrir el comportamiento de recuperación de PIN por correo cuando sea viable y restablecimiento asistido por Dinamizador sin revelar al Dinamizador el PIN actual ni el nuevo.
9. Integrar el nuevo flujo con el control de estación y la asignación de sesiones existentes, evitando modelos o protocolos paralelos incompatibles.

### No incluye

- Implementar la futura Página Web Nacional, su interfaz final, reportes finales o reglas regionales aún no definidas.
- Registrar usuarios nuevos desde Usuario PC ni redefinir el ciclo completo de alta, edición o baja de usuarios.
- Implementar en esta fase tablas, migraciones, transporte WSS/mTLS, mensajes de protocolo o acciones de producto; la arquitectura técnica aprobada se congela en los artefactos de diseño/especificación.
- Rediseñar por completo el motor de bloqueo, los hooks de kiosco o el control del sistema operativo que ya sean reutilizables.
- Resolver en este cambio cobros, tarifas, jornadas completas, transferencia de sesiones, extensión/reducción de tiempo o cierre de sesión, salvo los puntos mínimos necesarios para no crear estados incompatibles.
- Reemplazar de inmediato el flujo manual del Dinamizador; deberá mantenerse como contención/transición hasta validar el nuevo flujo.
- Inventar políticas de negocio, permisos, umbrales o reglas de conciliación no expresadas en los documentos normativos o aprobadas posteriormente.

## Áreas afectadas

| Área | Impacto esperado |
| --- | --- |
| `Soft_Usuario_PC/agente-infoplaza` — pantalla de bloqueo/login | Entradas de identidad y PIN, motivos, selección condicional de actividad, estados de error/espera/offline y recuperación de PIN. |
| Usuario PC — sesión y temporizador | Creación durable de sesión, modalidad explícita, actualización de estado y widget no invasivo. |
| Usuario PC — datos locales | Usuarios sincronizados, credenciales protegidas, catálogos necesarios, sesiones, transacciones pendientes y auditoría. |
| Comunicación LAN/WSS | WSS sobre TLS 1.3 con mTLS para tráfico privilegiado confiable, negociación v2, solicitud/respuesta, eventos y recuperación de estado; el parser legado no concede autorización. |
| `Soft_Dinamizador` — sesiones | Autorización de solicitudes, convivencia con asignación manual y una única interpretación del estado de sesión. |
| `Soft_Dinamizador` — configuración y asistencia | Duración predeterminada, política de modalidad, estado de estaciones y restablecimiento asistido de PIN. |
| Sincronización central futura | Contratos de identidad, procedencia, modalidad, estado de sincronización y trazabilidad, sin implementar la web. |
| Especificaciones y pruebas posteriores | Escenarios funcionales, seguridad, desconexión, reinicio, idempotencia y compatibilidad; la implementación posterior estará sujeta a TDD estricto. |

## Guardrails de integración

1. **Autoridad normativa:** Usuario PC se valida contra su documento rector y el Dinamizador contra el suyo. Cualquier contradicción de interacción es un bloqueo, no una decisión implícita del código existente.
2. **Evolución incremental:** aplicar `KEEP > EXTEND > REFACTOR > REPLACE > ADD`; no efectuar reescrituras ni crear implementaciones paralelas por conveniencia.
3. **Identidad estable:** usuarios, estaciones, sesiones, actividades y transacciones sincronizables usarán identificadores internos estables. Cédula, pasaporte, MAC y nombre de PC no serán claves técnicas primarias.
4. **Offline-first:** autenticación local elegible, inicio básico permitido por política, persistencia de pendientes, reintentos y recuperación de estado no dependerán de Internet permanente.
5. **LAN como ruta operativa segura:** la coordinación privilegiada Dinamizador ↔ Usuario PC funcionará directamente mediante WSS sobre TLS 1.3 con mTLS; la nube no será requisito para autenticar el vínculo en runtime y WS en claro no será ruta confiable.
6. **Durabilidad antes de habilitación:** una estación no se considerará en sesión si el registro local crítico no quedó confirmado. Las operaciones repetidas no deberán crear sesiones duplicadas.
7. **Modalidad explícita:** cada sesión guardará la modalidad utilizada; no se inferirá después desde logs o conectividad incidental.
8. **Credenciales protegidas:** el PIN no se persistirá en texto plano. El Dinamizador no conocerá el PIN actual ni el nuevo durante un restablecimiento.
9. **Auditoría trazable:** los eventos críticos incluirán identidad interna, estación, sesión cuando exista, tiempo, origen, modalidad y resultado, respetando la política de privacidad que se apruebe.
10. **Catálogos y conciliación con autoridad clara:** antes de implementar motivos, actividades, usuarios o sesiones sincronizables se definirá autoridad, vigencia, conflicto, procedencia y deduplicación.
11. **Transición compatible:** la ampliación del protocolo y del modelo de sesión deberá permitir despliegue gradual y convivencia controlada con estaciones/Dinamizadores aún no actualizados.

## Estrategia esperada: KEEP / EXTEND / REFACTOR / ADD

| Clasificación | Elementos | Dirección propuesta |
| --- | --- | --- |
| **KEEP** | Máquina de estados de bloqueo/desbloqueo y hooks de kiosco que ya cumplen su función. | Conservarlos como mecanismos de control; el nuevo login deberá invocarlos sin duplicarlos. |
| **EXTEND** | Base WebSocket/LAN, temporizador y superficies flotantes reutilizables; repositorios y modales de sesión del Dinamizador. | Añadir capacidades de solicitud, confirmación, estado y datos normativos, preservando lo que ya funciona. |
| **REFACTOR** | Protocolo predominantemente server-push y flujo de sesión concentrado en asignación manual del Dinamizador. | Reorganizar responsabilidades para admitir login originado en la estación, respuestas correlacionadas y conciliación, sin crear un segundo modelo de sesión. |
| **ADD** | Formulario de login, motivos/actividades, autenticación local protegida, persistencia local de usuarios/catálogos/sesiones, cola offline, auditoría, modalidades y recuperación de PIN. | Incorporar únicamente las capacidades inexistentes y conectarlas con los mecanismos conservados/extendidos. |

No se anticipa **REPLACE** en esta propuesta. Si el diseño concluye que algún elemento debe reemplazarse, requerirá evidencia de incompatibilidad normativa y aprobación explícita.

## Riesgos y tradeoffs

| Riesgo | Impacto | Contención propuesta |
| --- | --- | --- |
| Ambigüedad en las transiciones entre modalidad autorizada y autónoma | Acceso indebido o denegación durante fallas LAN. | Bloquear diseño hasta aprobar umbrales, vigencia y condiciones de transición. |
| Credenciales sincronizadas obsoletas o usuario deshabilitado mientras la estación está offline | Acceso no autorizado o rechazo de un usuario válido. | Definir vigencia, revocación, tolerancia offline y política de riesgo antes de implementar. |
| Creación duplicada de sesiones por reintentos, reinicios o respuestas tardías | Tiempo y reportería inconsistentes. | Exigir identidad estable, correlación e idempotencia como criterio del diseño posterior. |
| Dos fuentes de verdad entre Usuario PC y Dinamizador | Estados de estación/sesión divergentes. | Aprobar autoridad y conciliación; reutilizar el modelo existente en vez de crear otro. |
| PIN de cuatro dígitos expuesto a fuerza bruta o recuperación insegura | Compromiso de cuentas y datos personales. | Aprobar protección, límites, bloqueo, verificación de identidad y auditoría antes del diseño. |
| Catálogos de motivos/actividades desactualizados | Clasificación incorrecta de visitas o UX confusa. | Definir autoridad, vigencia y comportamiento con datos parciales/offline. |
| Alcance demasiado amplio al incluir login, sesión, offline y recuperación | Entrega difícil de revisar y mayor riesgo operativo. | Diseñar después en incrementos verificables bajo una única propuesta, sin relajar criterios de consistencia. |
| Compatibilidad con versiones desplegadas | Interrupción de estaciones durante adopción gradual. | Mantener transición versionada y reversión al flujo manual hasta validar compatibilidad. |
| Auditoría con exceso de datos documentales | Riesgo de privacidad y soporte. | Definir minimización, acceso y retención antes de persistir eventos finales. |

El tradeoff principal es equilibrar continuidad offline con control de acceso vigente: permitir demasiado tiempo con datos obsoletos aumenta el riesgo de seguridad; exigir comunicación constante incumple offline-first y eleva el costo operativo.

## Rollback y contención

- Mantener disponible el flujo manual/autorizado existente del Dinamizador durante el despliegue gradual.
- Permitir desactivar el nuevo login por estación o Infoplaza y volver a estado bloqueado sin borrar sesiones, transacciones o auditoría ya registradas.
- Conservar compatibilidad temporal con los comandos actuales mientras se valida la extensión del contrato LAN.
- Si falla la creación local de una sesión, no desbloquear la estación y registrar el fallo para soporte.
- Si falla la sincronización posterior, conservar el registro como pendiente; el rollback funcional no debe eliminar datos no conciliados.
- Ante estado ambiguo después de reinicio o pérdida de comunicación, contener la estación y presentar el caso al mecanismo de recuperación/conciliación aprobado, en vez de iniciar otra sesión.
- Evitar rollback destructivo de datos. Cualquier reversión de esquema o protocolo se definirá en diseño con una ruta de compatibilidad y recuperación verificable.

## Criterios de éxito

1. Un usuario registrado, elegible y previamente sincronizado puede autenticarse desde la estación con cédula/pasaporte y PIN de cuatro dígitos sin depender de Internet permanente.
2. Todo intento exige uno o más motivos; capacitación/actividad presenta opciones programadas disponibles y, cuando no existen, permite continuar de forma explícita.
3. Las modalidades automática, autorizada y autónoma se comportan según la política aprobada y quedan registradas en cada sesión.
4. Un login exitoso crea exactamente una sesión durable con duración predeterminada configurable antes de habilitar la estación.
5. El estado de la estación y el widget muestran una sesión coherente sin interferir de manera invasiva con el uso del equipo.
6. Una desconexión o reinicio no pierde sesiones, transacciones ni auditoría; los pendientes pueden reintentarse y conciliarse sin duplicados.
7. Se auditan éxitos/fallos de login, inicio y modalidad de sesión, pérdida/recuperación de comunicación y cambios/restablecimientos de PIN.
8. El PIN no queda en texto plano y el Dinamizador no conoce el PIN actual ni el nuevo durante la asistencia.
9. Las relaciones se basan en IDs internos estables; cédula/pasaporte se usa solo como identidad documental o criterio de búsqueda.
10. El flujo manual y el control de bloqueo existentes permanecen operativos durante la transición y pueden contener un despliegue fallido.
11. Antes de implementar, las especificaciones derivadas cubren casos normales, offline, permisos/inelegibilidad, fallos parciales, respuestas tardías, reinicio y recuperación; la implementación posterior seguirá TDD estricto.

## OPEN REQUIREMENTS — bloquean diseño e implementación

Los siguientes puntos no deben resolverse silenciosamente en diseño ni código:

1. **Política de modalidad:** quién configura la modalidad por estación/Infoplaza; cuándo se considera temporal una falla; tiempo de gracia; vigencia de una autorización previa; y condiciones exactas para entrar/salir de modo autónomo.
2. **Contrato de autorización:** estados y resultados que puede emitir el Dinamizador, tiempo de espera, cancelación, rechazo, respuesta tardía y conducta cuando la comunicación se recupera durante el login.
3. **Elegibilidad offline:** antigüedad máxima de datos de usuario/credencial, tratamiento de usuarios suspendidos o eliminados, revocación, vencimiento y conducta si no existe un registro local confiable.
4. **Seguridad del PIN:** representación protegida aprobada, límites de intentos, ventana y duración de bloqueo, escalamiento, mensajes al usuario y política ante intentos distribuidos en varias estaciones.
5. **Recuperación/restablecimiento de PIN:** verificación de identidad, cuándo se permite correo, contenido/vigencia del mecanismo enviado, procedimiento asistido, permisos del Dinamizador y resultado cuando no hay Internet.
6. **Motivos de visita:** catálogo autoritativo, cardinalidad y combinaciones permitidas, obligatoriedad real, opción “otro”, vigencia offline y qué valor se registra ante catálogo faltante o parcial.
7. **Capacitaciones/actividades:** autoridad del catálogo, ventana de fechas, filtros por Infoplaza/estación/usuario, selección de una o varias, cupos/inscripción y semántica exacta de “no existen programadas”.
8. **Reglas de sesión:** duración predeterminada y sus niveles de configuración; una sesión activa por usuario/estación; conducta si el usuario ya tiene sesión; y recuperación tras cierre abrupto, reinicio o pérdida de energía.
9. **Autoridad y conciliación:** fuente de verdad para usuario, estación, sesión, motivos y actividades; deduplicación; orden de eventos; resolución de conflictos; retención de pendientes; y tratamiento de rechazos centrales.
10. **Privacidad y auditoría:** datos documentales que pueden registrarse, minimización/enmascarado, retención, acceso, exportación y respuesta operativa a eventos de seguridad.
11. **Contenido del widget:** campos visibles, acciones permitidas, avisos de tiempo y comportamiento en estados degradados, para asegurar que sea útil y no invasivo.
12. **Compatibilidad normativa cruzada:** resolución explícita de cualquier diferencia entre el flujo originado en Usuario PC y las reglas de autorización/control establecidas para Dinamizador.

## Proposal question round

Esta ronda se incluye porque la ejecución es automática y no hubo una pausa interactiva. Las preguntas buscan mejorar el PRD/propuesta descubriendo reglas de negocio, impacto, casos límite y tradeoffs; el responsable de producto puede responderlas, omitirlas, corregir el enfoque o solicitar una segunda ronda.

1. ¿Quién decide la modalidad de cada estación y qué límite temporal exacto permite pasar de “autorizada” a “autónoma” sin convertir una falla de comunicación en acceso indefinido?
2. ¿Qué usuarios siguen siendo elegibles offline y durante cuánto tiempo después de su última sincronización, especialmente si podrían haber sido suspendidos o haber cambiado su PIN?
3. ¿Qué reglas de negocio rigen los motivos y actividades: combinaciones válidas, una o varias actividades, cupos/inscripción y comportamiento con catálogo parcial o desactualizado?
4. ¿Qué debe ocurrir si el usuario o la estación ya tiene una sesión activa, o si llega tarde una autorización después de un reintento/reinicio?
5. ¿Qué verificación de identidad y límites de intentos deben proteger el PIN y su restablecimiento sin revelar credenciales al Dinamizador?

### Supuestos provisionales para revisión

- Cédula/pasaporte sirve para localizar a un usuario interno estable, no como clave primaria.
- Seleccionar al menos un motivo es obligatorio; la ausencia de actividades programadas no bloquea el login.
- La sesión se registra localmente antes de desbloquear y se sincroniza después de forma idempotente.
- El modo autónomo solo existe como degradación temporal y previamente autorizada, nunca como bypass permanente.
- El flujo manual del Dinamizador permanece como mecanismo de transición y contingencia.
- No se elegirá una política concreta para los requisitos abiertos hasta recibir aprobación de producto/seguridad/operación.

## Cierre PR-08 y siguiente paso recomendado

Los nueve bloqueadores técnicos originales de PR-08 están **CLOSED** y el contrato está listo para planificación de implementación. Los umbrellas PR-08A, PR-08B y PR-08U son no ejecutables y quedan superseded por PR-08A1/A2, PR-08B1/B2 y PR-08U1/U2. No se autoriza crear worktrees en este cierre; el primer worktree potencial es PR-08A1, únicamente después de un preflight de implementación nuevo. PR-08B1 depende de los contratos A verificados e integrados; PR-08U1 puede iniciar después del freeze contractual A1/A2 y de aceptar el spike de almacenamiento protegido.

Permanecen abiertos los requisitos de negocio no relacionados de esta propuesta —modalidad/autonomía, vigencia offline, PIN, catálogos, sesión, conciliación, privacidad/widget y compatibilidad normativa—. El cierre técnico PR-08 no los resuelve ni autoriza acciones de producto.
