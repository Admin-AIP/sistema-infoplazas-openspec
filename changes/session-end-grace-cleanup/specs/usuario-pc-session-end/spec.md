# Especificación de fin de sesión de Usuario PC

## Propósito

Definir el comportamiento funcional para advertir el vencimiento, conceder una única gracia válida, terminar el acceso, limpiar los residuos de la sesión y habilitar la reutilización segura de una estación Usuario PC. La estación solo queda preparada/`AVAILABLE` después de revocar el acceso, completar la limpieza y verificar un resultado seguro; ante fallo o incertidumbre, el comportamiento es cerrado y contenido.

Esta especificación no selecciona mecanismos técnicos de limpieza, no define tarifas ni cálculos comerciales adicionales y no altera ni absorbe el alcance de recuperación de PR-06.

## Semántica funcional de estados

Los nombres siguientes son conceptos observables y no exigen nombres exactos de enums, campos persistidos o eventos:

| Concepto | Semántica requerida |
| --- | --- |
| Sesión activa | El tiempo normal no ha terminado y el usuario conserva el acceso autorizado. |
| Gracia activa | El tiempo normal terminó, existe una gracia válida ya concedida y el usuario conserva excepcionalmente acceso completo hasta terminarla. |
| Sesión terminada | Ya no existe tiempo normal ni gracia válida que autorice acceso del usuario. |
| Estación bloqueada | El acceso interactivo del usuario fue revocado antes de cualquier cierre o limpieza lenta. |
| Limpieza en curso | La estación permanece bloqueada mientras se cierran aplicaciones, se tratan residuos y se verifica seguridad. |
| Limpieza fallida | La limpieza o su verificación falló, quedó incompleta o produjo un resultado desconocido; la estación permanece contenida y no disponible. |
| Preparada/`AVAILABLE` | El fin, la revocación de acceso, la limpieza y la verificación segura se completaron satisfactoriamente y la estación puede recibir al siguiente usuario. |

## Requisitos

### Requirement: Convergencia de las terminaciones normales

Todas las causas normales de terminación —vencimiento del tiempo, finalización voluntaria del usuario, orden de finalización del Dinamizador y otras causas normales autorizadas— MUST converger, cuando la terminación se haga efectiva, en una única frontera segura de fin y limpieza. Esa frontera MUST revocar el acceso, bloquear la estación, registrar el fin, ejecutar la limpieza, verificar su resultado y solo entonces permitir `AVAILABLE`.

La convergencia MUST NOT conceder una gracia implícita a ninguna causa. La política de advertencia y gracia aplicable específicamente a una terminación ordenada por el Dinamizador permanece abierta.

#### Scenario: Vencimiento normal sin gracia

- GIVEN una sesión activa cuyo tiempo normal llega a `00:00` sin una gracia válida
- WHEN se hace efectivo el vencimiento
- THEN el sistema MUST terminar el acceso y entrar inmediatamente en la frontera segura común
- AND MUST bloquear la estación antes de iniciar cierre o limpieza lenta

#### Scenario: Finalización voluntaria

- GIVEN una sesión activa o una gracia activa
- WHEN el usuario ejecuta la acción de finalizar
- THEN el sistema MUST terminar el acceso y entrar en la misma frontera segura común
- AND MUST NOT conceder ni renovar una gracia por esa causa

#### Scenario: Finalización ordenada por el Dinamizador

- GIVEN una sesión que recibe una orden autorizada de finalización del Dinamizador
- WHEN la terminación se hace efectiva conforme a la política pendiente de advertencia y gracia
- THEN el sistema MUST aplicar la misma frontera segura común
- AND MUST NOT inferir una política de advertencia, interrupción o concesión de gracia todavía no aprobada

#### Scenario: Otra causa normal autorizada

- GIVEN una sesión afectada por una causa normal de terminación distinta de las anteriores
- WHEN esa causa termina el acceso
- THEN el sistema MUST aplicar la misma frontera segura común
- AND MUST NOT crear una gracia no solicitada y aprobada

### Requirement: Advertencias configurables y decisión previa de gracia

El sistema MUST admitir advertencias de pre-vencimiento configurables. Cuando la advertencia final esté habilitada, MUST presentarse conceptualmente a un minuto del vencimiento normal y MUST preguntar conductualmente si existen trabajos o archivos pendientes que deban finalizarse o guardarse. El texto final, su accesibilidad y sus idiomas permanecen abiertos.

Solo una respuesta afirmativa explícita recibida antes de `00:00` MAY solicitar la gracia. Una respuesta negativa o la ausencia de respuesta antes de `00:00` MUST NOT conceder gracia ni interpretarse como consentimiento.

#### Scenario: Advertencia final habilitada

- GIVEN una sesión activa con la advertencia final habilitada
- WHEN resta conceptualmente un minuto para el vencimiento normal
- THEN el sistema MUST advertir el vencimiento
- AND MUST preguntar si el usuario necesita finalizar o guardar trabajo o archivos pendientes
- AND MAY usar el copy aprobado posteriormente sin cambiar esta conducta

#### Scenario: Advertencia final deshabilitada

- GIVEN una política que deshabilita la advertencia final
- WHEN se aproxima el vencimiento normal
- THEN el sistema MUST NOT depender de esa advertencia para terminar el tiempo normal con seguridad
- AND MUST NOT conceder gracia automáticamente

#### Scenario: Respuesta afirmativa antes del vencimiento

- GIVEN que la pregunta de gracia está vigente y la sesión aún no llegó a `00:00`
- WHEN el usuario responde afirmativamente
- THEN el sistema MUST registrar la solicitud como hecha antes del vencimiento
- AND MUST preparar una única gracia configurable para comenzar en `00:00`

#### Scenario: Respuesta negativa antes del vencimiento

- GIVEN que la pregunta de gracia está vigente
- WHEN el usuario responde NO antes de `00:00`
- THEN el sistema MUST registrar que no se concedió gracia
- AND MUST terminar el acceso y bloquear inmediatamente al llegar a `00:00`

#### Scenario: Falta de respuesta al llegar a cero

- GIVEN que la pregunta de gracia fue presentada pero no existe respuesta afirmativa válida
- WHEN el tiempo normal llega a `00:00`
- THEN el sistema MUST tratar la falta de respuesta como ausencia de gracia
- AND MUST terminar el acceso y bloquear inmediatamente
- AND MUST NOT conceder gracia por demora de UI, reintento o respuesta tardía

### Requirement: Gracia única de acceso

Cuando exista una respuesta afirmativa válida, el tiempo normal y facturable MUST terminar en el `00:00` normal y exactamente una gracia de duración configurable MUST comenzar en ese instante. Durante la gracia, el sistema MUST mantener acceso completo al escritorio, MUST mostrar el tiempo restante y MUST comunicar que su propósito es finalizar o guardar trabajo y cerrar la sesión.

La gracia MUST ser no facturable, MUST concederse como máximo una vez por sesión, MUST NOT encadenarse, renovarse ni reiniciarse, y MAY terminar anticipadamente por acción del usuario. Un reintento, reinicio, demora de interfaz o cambio incidental de conectividad MUST NOT crear una segunda gracia.

#### Scenario: Inicio de gracia aceptada

- GIVEN una respuesta afirmativa válida recibida antes de `00:00`
- WHEN el tiempo normal llega a `00:00`
- THEN el tiempo normal y facturable MUST terminar
- AND exactamente una gracia configurable MUST comenzar
- AND el usuario MUST conservar acceso completo al escritorio
- AND el tiempo restante de gracia MUST ser visible
- AND la limpieza MUST NOT comenzar mientras la gracia siga activa

#### Scenario: Vencimiento de la gracia

- GIVEN una gracia activa
- WHEN su tiempo restante llega a cero
- THEN el sistema MUST terminar inmediatamente el acceso
- AND MUST bloquear la estación antes de registrar, limpiar y verificar
- AND MUST NOT ofrecer ni iniciar otra gracia

#### Scenario: Fin anticipado de la gracia

- GIVEN una gracia activa con tiempo restante
- WHEN el usuario ejecuta la acción de finalizar
- THEN el sistema MUST terminar anticipadamente la gracia
- AND MUST usar únicamente la duración efectiva transcurrida
- AND MUST bloquear inmediatamente y continuar con la frontera segura común

#### Scenario: Intento de segunda gracia

- GIVEN que la sesión ya recibió, consumió o terminó anticipadamente su única gracia
- WHEN ocurre un reintento, reinicio, demora de UI, cambio incidental de conectividad o una nueva solicitud
- THEN el sistema MUST NOT encadenar, renovar, reiniciar ni conceder otra gracia

### Requirement: Revocación de acceso antes de la limpieza

Excepto mientras exista una gracia activa válida, el sistema MUST terminar el acceso del usuario y bloquear la estación antes de cualquier cierre de aplicaciones, limpieza o verificación que pueda demorar. El cierre de una aplicación MUST NOT dejar al usuario operando la estación. La estación MUST permanecer bloqueada durante el registro, la limpieza, la verificación, la contención y cualquier recuperación posterior.

#### Scenario: Limpieza lenta tras el fin de acceso

- GIVEN una sesión terminada sin gracia activa
- WHEN el cierre o la limpieza requiere tiempo adicional
- THEN la estación MUST estar bloqueada antes de iniciar ese trabajo
- AND el usuario MUST NOT conservar acceso interactivo durante la espera

#### Scenario: Aplicación cerrada con otras superficies disponibles

- GIVEN que terminó el acceso del usuario
- WHEN se cierra una aplicación pero aún quedan otras superficies o procesos de usuario
- THEN el sistema MUST mantener bloqueada la estación
- AND MUST NOT considerar el cierre de esa aplicación como terminación segura ni permitir operación del usuario

### Requirement: Cierre escalonado y protección de procesos

El sistema MUST solicitar primero el cierre controlado de las aplicaciones. Después MUST esperar el timeout configurable de política y solo MAY forzar el cierre de aplicaciones administradas que sean explícitamente elegibles y continúen activas. El sistema MUST NOT ejecutar `kill-all` ni terminación indiscriminada.

La política MUST proteger, como mínimo, los procesos de Usuario PC, componentes esenciales de Windows, servicios de agentes, aplicaciones institucionales que deban persistir y cualquier proceso cuya terminación pueda desestabilizar la estación. Un proceso protegido o no clasificable MUST NOT forzarse para acelerar `AVAILABLE`; si impide completar o verificar la limpieza, el resultado MUST tratarse como fallo o necesidad de atención.

La duración y autoridad del timeout, la lista concreta de procesos y sus excepciones permanecen abiertas.

#### Scenario: Cierre controlado exitoso

- GIVEN aplicaciones de usuario abiertas al terminar el acceso
- WHEN comienza la etapa de cierre
- THEN el sistema MUST solicitar primero su cierre controlado
- AND MUST continuar a limpieza sin cierre forzado si las aplicaciones terminan dentro del timeout de política

#### Scenario: Aplicación administrada elegible no responde

- GIVEN una aplicación administrada explícitamente elegible que no termina tras la solicitud controlada
- WHEN vence el timeout de política
- THEN el sistema MAY forzar únicamente el cierre de esa aplicación elegible
- AND MUST registrar el resultado sin incluir contenido de usuario

#### Scenario: Seguridad de proceso protegido

- GIVEN un proceso de Usuario PC, esencial de Windows, de agente, institucional persistente o potencialmente desestabilizador
- WHEN continúa activo después del timeout de cierre controlado
- THEN el sistema MUST NOT terminarlo indiscriminadamente
- AND MUST mantener la estación bloqueada y fuera de `AVAILABLE` si su presencia impide demostrar una limpieza segura

#### Scenario: Prohibición de cierre masivo

- GIVEN múltiples procesos activos al terminar una sesión
- WHEN se requiere cerrar aplicaciones de usuario
- THEN el sistema MUST evaluar solo aplicaciones administradas elegibles conforme a política
- AND MUST NOT usar una operación general de tipo `kill-all`

### Requirement: Limpieza integral de privacidad

Cerrar aplicaciones MUST NOT considerarse limpieza suficiente. La limpieza y su verificación MUST cubrir, según la política técnica que se apruebe, sesiones de navegador, cookies, historial, caché, formularios y autocompletado, credenciales, descargas, archivos temporales, sesiones web persistentes, documentos y otros residuos personales accesibles.

El resultado normativo MUST impedir que el siguiente usuario acceda a datos, cuentas, documentos o sesiones del usuario anterior. La estrategia técnica concreta y la política institucional de archivos locales permanecen abiertas.

#### Scenario: Navegador cerrado con residuos persistentes

- GIVEN que los procesos y ventanas del navegador fueron cerrados
- WHEN aún existen cookies, historial, caché, formularios, credenciales, descargas o sesiones web reutilizables
- THEN el sistema MUST considerar incompleta la limpieza
- AND MUST mantener la estación bloqueada y fuera de `AVAILABLE`

#### Scenario: Residuos documentales o personales

- GIVEN que terminó una sesión
- WHEN la verificación detecta documentos, temporales, descargas u otros residuos personales accesibles al siguiente usuario
- THEN la verificación MUST fallar
- AND la estación MUST permanecer contenida hasta completar una recuperación segura

#### Scenario: Verificación de privacidad satisfactoria

- GIVEN que finalizaron el acceso, el cierre y la limpieza
- WHEN la verificación aprobada demuestra que el siguiente usuario no puede acceder a datos, cuentas, documentos ni sesiones anteriores
- THEN el objetivo de privacidad MUST considerarse satisfecho para esa ejecución

### Requirement: Disponibilidad verificada y fallo cerrado

El sistema MUST declarar la estación preparada/`AVAILABLE` únicamente después de confirmar el fin de la sesión, la revocación del acceso, la finalización de la limpieza y una verificación segura satisfactoria. Un timeout, la ausencia de telemetría, el cierre de una sola aplicación o un resultado desconocido MUST NOT bastar para declarar `AVAILABLE`.

Si el bloqueo, el cierre, la limpieza o la verificación falla, queda incompleto o no puede demostrarse, el sistema MUST mantener la estación bloqueada, contenida y fuera de `AVAILABLE`, en un estado de atención o recuperación.

#### Scenario: Transición segura a AVAILABLE

- GIVEN una sesión terminada y una estación bloqueada
- WHEN la limpieza termina y la verificación aprobada confirma un resultado seguro
- THEN el sistema MUST marcar la estación preparada/`AVAILABLE`
- AND solo entonces MAY presentar la bienvenida al siguiente usuario

#### Scenario: Fallo de limpieza

- GIVEN una estación bloqueada con limpieza en curso
- WHEN la limpieza falla o queda incompleta
- THEN el sistema MUST registrar el fallo
- AND MUST mantener la estación bloqueada o contenida para atención o recuperación
- AND MUST NOT pasar automáticamente a `AVAILABLE`

#### Scenario: Resultado desconocido o falta de evidencia

- GIVEN que no existe evidencia suficiente del resultado de limpieza o verificación
- WHEN el sistema evalúa la disponibilidad
- THEN MUST tratar el resultado desconocido como no seguro
- AND MUST NOT inferir `AVAILABLE` por timeout o ausencia de telemetría

### Requirement: Tiempos y ocupación sin cálculo comercial adicional

El sistema MUST preservar por separado el tiempo normal, la duración de gracia concedida y la duración de gracia efectivamente usada. La ocupación de estación MUST equivaler exactamente a la duración normal de la sesión más la duración efectiva de gracia. La latencia posterior de bloqueo, cierre, limpieza, verificación o recuperación MUST permanecer separada de la ocupación.

Esta especificación MUST NOT definir tarifas, recargos ni cálculos comerciales adicionales. La semántica, destinatarios y visualización del reporte de latencia de limpieza/disponibilidad permanecen abiertos.

#### Scenario: Gracia consumida parcialmente

- GIVEN una sesión con tiempo normal usado y una gracia terminada anticipadamente
- WHEN se calcula la ocupación de estación
- THEN la ocupación MUST sumar el tiempo normal y solo la gracia efectivamente usada
- AND MUST NOT sumar la gracia concedida pero no usada ni la latencia de limpieza

#### Scenario: Limpieza prolongada

- GIVEN que el acceso terminó y la limpieza tarda más que lo esperado
- WHEN se informa la ocupación
- THEN la latencia de limpieza MUST permanecer separada
- AND MUST NOT ampliar el tiempo normal, la gracia efectiva ni la ocupación

### Requirement: Trazabilidad minimizada del fin y la limpieza

El sistema MUST permitir reconstruir el fin mediante identificadores internos, tiempos, estación, sesión, causa, transiciones y resultados no sensibles. Como mínimo, MUST registrar equivalentes funcionales de: hora de finalización, motivo, tiempo normal usado y estado final de sesión; y, cuando corresponda, `grace_requested`, `grace_started`, `grace_duration_granted`, `grace_duration_used` y `grace_finished_early`.

La planificación técnica MUST contemplar eventos equivalentes a `cleanup_started`, `application_close_failed`, `forced_close`, `cleanup_completed` y `cleanup_failed`, además de la evidencia necesaria de bloqueo, verificación, contención o recuperación. Los nombres exactos permanecen como decisión técnica.

La trazabilidad MUST NOT registrar contenido de documentos o aplicaciones, credenciales, PIN, datos sensibles, texto de formularios, archivos, URLs, páginas visitadas ni cuentas del usuario.

#### Scenario: Fin normal con gracia anticipadamente terminada

- GIVEN una gracia solicitada, iniciada y terminada anticipadamente
- WHEN se registra el fin
- THEN la trazabilidad MUST incluir la causa, hora de finalización, tiempo normal usado, estado final y equivalentes de solicitud, inicio, duración concedida, duración usada y fin anticipado
- AND MUST NOT incluir contenido de usuario

#### Scenario: Cierre forzado elegible y limpieza completada

- GIVEN que una aplicación elegible requirió cierre forzado y la limpieza terminó satisfactoriamente
- WHEN se registra la secuencia
- THEN la trazabilidad MUST distinguir el inicio de limpieza, el fallo de cierre controlado, el cierre forzado y la finalización de limpieza
- AND MUST usar resultados y códigos no sensibles

#### Scenario: Registro de limpieza fallida

- GIVEN que la limpieza o su verificación falla
- WHEN se registra el resultado
- THEN la trazabilidad MUST permitir distinguir el fallo y el estado final contenido
- AND MUST NOT capturar documentos, credenciales, PIN, páginas, cuentas ni otro contenido sensible

### Requirement: Spike obligatorio en Windows real

Antes de seleccionar la estrategia técnica final, el cambio MUST ejecutar un spike en una estación Windows real. El spike MUST comparar: cierre controlado de aplicaciones; cierre más limpieza de navegador o perfil; logoff controlado de Windows o reset de perfil; y combinaciones de estas alternativas.

La comparación MUST evaluar privacidad, datos residuales, estabilidad de Windows, recuperación, compatibilidad con aplicaciones institucionales y procesos protegidos, reinicio automático de agentes, experiencia de usuario, duración, efectos sobre trabajo no guardado y capacidad de volver de forma segura a `AVAILABLE`. Una simulación o una prueba unitaria por sí sola MUST NOT satisfacer este requisito.

Esta especificación MUST NOT elegir anticipadamente una de las estrategias comparadas.

#### Scenario: Evidencia comparativa válida

- GIVEN que aún no se ha aprobado una estrategia técnica final
- WHEN se ejecuta el spike en una estación Windows real
- THEN MUST probarse cada alternativa y combinaciones relevantes contra todos los criterios definidos
- AND MUST recopilarse evidencia de residuos y de retorno seguro a `AVAILABLE`

#### Scenario: Evidencia insuficiente

- GIVEN resultados obtenidos solo mediante simulación o pruebas unitarias
- WHEN se intenta seleccionar la estrategia final
- THEN esos resultados MUST NOT considerarse cumplimiento del spike obligatorio
- AND la decisión técnica MUST permanecer abierta

### Requirement: Interrupciones y recuperación futura

El diseño y la planificación posteriores MUST analizar como casos excepcionales futuros la pérdida de energía, crash, reinicio de Windows, pérdida de comunicación y limpieza interrumpida, además de fallos de procesos, perfil o almacenamiento. El análisis MUST definir contención, detección, conciliación y condiciones de recuperación sin alterar, redefinir ni absorber PR-06.

Mientras el resultado permanezca incompleto o desconocido, la estación MUST permanecer bloqueada, contenida y fuera de `AVAILABLE`. La estrategia durable de recuperación permanece abierta.

#### Scenario: Interrupción durante limpieza

- GIVEN una estación bloqueada con limpieza en curso
- WHEN ocurre una pérdida de energía, crash, reinicio de Windows, pérdida de comunicación u otra interrupción
- THEN el resultado MUST considerarse incompleto o desconocido
- AND la estación MUST NOT volver automáticamente a `AVAILABLE` sin recuperación y verificación seguras

#### Scenario: Separación respecto de PR-06

- GIVEN que se diseña la recuperación de una limpieza interrumpida
- WHEN se definen detección, reintento, contención o conciliación
- THEN el diseño MUST conservar intacto el contrato y alcance de PR-06
- AND MUST NOT trasladar estos casos al alcance de PR-06 ni absorber su recuperación existente

## OPEN REQUIREMENTS — alcance preservado

La especificación funcional, el diseño y la planificación no dependiente MAY avanzar, pero los siguientes puntos MUST permanecer abiertos hasta su resolución explícita. Cada punto bloquea únicamente su decisión técnica definitiva, la implementación productiva correspondiente y las unidades de trabajo que dependan de ella; MUST NOT bloquear el spike Windows ya autorizado.

1. **Política Dinamizador ante terminación remota:** definir exclusivamente qué ocurre con las advertencias y con la gracia cuando el Dinamizador ordena terminar remotamente una sesión: si se muestran, se interrumpen o se permiten y cuál es su precedencia respecto de esa orden. No reabre el contrato confirmado de advertencias configurables, advertencia final ni gracia solicitada antes del vencimiento normal.
2. **Estrategia final de limpieza:** decisión basada en el spike entre cierre controlado, limpieza de navegador/perfil, logoff/reset de perfil Windows o combinaciones; evidencias requeridas para declarar reutilización segura y comportamiento de cada alternativa.
3. **Timeout de cierre controlado:** duración, autoridad de configuración, observabilidad y conducta tras su vencimiento antes de la fuerza elegible.
4. **Lista de procesos protegidos:** procesos/clases protegidas, autoridad de la lista, excepciones, actualización, relación con accesibilidad/seguridad/administración y comportamiento cuando impiden finalizar limpieza.
5. **Mecanismo de limpieza de navegador y perfil:** navegadores soportados, datos a tratar, perfiles, descargas, cachés, cookies, credenciales, temporales, verificación y límites de seguridad/privacidad.
6. **Documentos no guardados después de gracia:** aviso, oportunidad de guardar, alcance de preservación o descarte, prioridad frente al timeout y resultado cuando la aplicación no responde.
7. **Recuperación de limpieza interrumpida:** estado durable mínimo, detección al reiniciar, reintento o contención, intervención de soporte, conciliación y condiciones para volver a AVAILABLE; sin absorber ni redefinir PR-06.
8. **Copy final de UI:** textos, accesibilidad, idiomas, aviso de vencimiento, pregunta/respuesta de gracia, explicación de cierre y comunicación de estados de contención.
9. **Reporte final:** destinatarios, retención, visualización y semántica aprobada de tiempo normal, gracia, ocupación —ya definida como normal más gracia efectiva—, resultado de limpieza, latencia de limpieza/disponibilidad e incidencias, sin introducir tarifas o cálculos comerciales.
10. **Política institucional de archivos locales de usuario:** qué archivos pueden persistir, eliminarse, resguardarse o requerir intervención; responsabilidades, retención y cumplimiento aplicable.

#### Scenario: Decisión todavía no aprobada

- GIVEN una implementación o unidad de trabajo que depende de cualquiera de los diez OPEN REQUIREMENTS
- WHEN no existe una resolución explícita y aprobada para ese punto
- THEN el sistema y su planificación MUST NOT inferir ni fijar la decisión técnica pendiente
- AND el trabajo no dependiente y el spike Windows MAY continuar dentro de los comportamientos ya confirmados
