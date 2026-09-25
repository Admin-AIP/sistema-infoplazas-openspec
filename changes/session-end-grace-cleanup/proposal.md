# Propuesta: cierre seguro de sesión, gracia y limpieza de estación

- **Cambio:** `session-end-grace-cleanup`
- **Estado:** propuesta preparada para especificación funcional, diseño y planificación de tareas. No autoriza cambios de código ni de contratos; los OPEN REQUIREMENTS solo condicionan las decisiones técnicas definitivas, la implementación productiva y las unidades de trabajo que dependan de cada una.
- **Ubicación en la secuencia:** posterior a la línea base verificada entre repositorios hasta PR-16 del cambio `session-start-login-usuario-pc`. No se asigna un número de PR a este cambio.
- **Dependencias:** fundamentos completos de activación y ciclo de vida de sesión, bloqueo físico, temporizador durable, enlace/contratos ya verificados y la prueba de compatibilidad, rollback y E2E de PR-16. Son dependencias de contrato entre repositorios, no de ancestría Git.

## Resultado normativo

Cuando termina el acceso de una sesión, la estación **se bloquea antes de cualquier limpieza lenta**, ejecuta y verifica la limpieza de privacidad, y solo puede pasar a **AVAILABLE/bienvenida** cuando se haya demostrado que es segura para reutilización. Si no puede demostrarse el bloqueo, el cierre, la limpieza o su verificación, la estación permanece bloqueada y fuera de AVAILABLE: el comportamiento es **fail-closed**.

El fin del tiempo normal no es necesariamente el fin del acceso: una gracia válida y solicitada antes del vencimiento permite continuidad excepcional de acceso. En ausencia de esa gracia, o al terminarla, aplica el límite seguro anterior. Este cambio no define tarifas, cobros ni cálculos comerciales adicionales.

## Intención y problema

La base de sesiones prevista hasta PR-16 habilita inicio, activación, recuperación y compatibilidad segura, pero no resuelve aún el final operativo de una sesión de estación pública. Sin una frontera de salida definida, el equipo puede quedar con aplicaciones, navegador, perfiles, descargas, credenciales o datos residuales; además, una limpieza lenta o fallida puede competir con la visibilidad de AVAILABLE y exponer la siguiente persona a información anterior.

La propuesta establece una experiencia predecible para quien usa la estación y una contención operativa para el Dinamizador: advertir con anticipación, permitir una única decisión de gracia bajo política, y asegurar que toda terminación llegue a una estación bloqueada, privada, trazable y segura para el siguiente uso.

## Alcance propuesto

### Incluye

1. Mantener advertencias previas configurables. Cuando esté habilitada, la advertencia final, conceptualmente situada a un minuto del vencimiento, pregunta si queda trabajo por realizar; su copy técnico final permanece abierto.
2. Permitir que una respuesta afirmativa antes del vencimiento normal habilite, como máximo, una gracia configurable. Una respuesta **NO** o la falta de respuesta antes de `00:00` significan que no hay gracia; no se autoriza ni se presupone otra acción de descarte de la UI.
3. Mantener una única gracia no facturable por sesión, configurable por política y solo habilitable mediante la respuesta afirmativa anterior. Si fue habilitada, comienza exactamente en el `00:00` normal y conserva acceso de escritorio completo durante el intervalo concedido. La única incertidumbre de política relativa al Dinamizador es cómo se aplican advertencia y gracia cuando este ordena una terminación remota.
4. Permitir terminar la gracia antes de tiempo por la acción de finalización correspondiente y llevar ese caso al mismo límite seguro de cierre. Después de consumir o rechazar la única gracia no se crea otra por reintento, reinicio, demora de UI o cambio incidental de conectividad.
5. Hacer que todas las causas normales de terminación —vencimiento sin gracia, finalización voluntaria, fin anticipado de gracia y terminación operativa autorizada— converjan en una sola frontera segura: bloquear primero, cerrar y limpiar después, verificar y recién entonces habilitar AVAILABLE/bienvenida. Esta convergencia no inventa gracia para causas que no la tienen aprobada.
6. Definir la ocupación de estación exactamente como la duración normal de sesión más la duración efectiva de gracia, sin mezclarla con tarifas ni con la latencia de limpieza/disponibilidad. Esta última podrá observarse separadamente en el futuro; la semántica, destinatarios y visualización del reporte permanecen abiertos.
7. Definir un cierre escalonado: solicitud de cierre controlado de aplicaciones, espera hasta un timeout de política y cierre forzado solo de procesos elegibles que continúen activos. Queda prohibido un enfoque de «kill-all» o terminación indiscriminada de procesos.
8. Diseñar una lista de procesos protegidos y sus reglas de preservación. La limpieza no puede sacrificar procesos del sistema, seguridad, accesibilidad, administración o recuperación solo para declarar rápido una estación disponible.
9. Cubrir privacidad residual de navegador y perfiles: ventanas/procesos, sesiones web, descargas, cachés, cookies, credenciales, archivos temporales y datos asociados que puedan quedar accesibles al siguiente usuario. Cerrar un navegador no constituye por sí mismo prueba suficiente de limpieza.
10. Registrar trazabilidad de la terminación, gracia, bloqueo, cierre, limpieza, verificación, recuperación y resultado, usando IDs internos, tiempos, estación, sesión, causa y códigos/resultados no sensibles. No se registrará contenido de documentos, archivos, URLs, credenciales, texto de formularios ni otro contenido personal.
11. Analizar interrupciones durante bloqueo, cierre o limpieza —reinicio, caída de energía, proceso no terminable, error de perfil, fallo de almacenamiento o recuperación incompleta— y definir cómo contener la estación y continuar/conciliar el trabajo pendiente sin exponer datos.
12. Ejecutar obligatoriamente un spike en una estación Windows real antes de escoger la estrategia técnica final. El spike es un prerrequisito de planificación ya autorizado, no una actividad bloqueada por elegir previamente dicha estrategia. Comparará evidencia de: cierre controlado; limpieza de navegador/perfil; cierre de sesión de Windows o reinicio de perfil; y combinaciones de esos mecanismos. Debe medir privacidad residual, compatibilidad con procesos protegidos, recuperación, duración, efectos sobre datos no guardados y capacidad de volver de forma segura a AVAILABLE. No basta una simulación o prueba unitaria.

### No incluye

- Alterar los contratos, el alcance o la recuperación de PR-06; esta propuesta consume la base verificada posterior a PR-16 y no absorbe la recuperación de arranque ya definida.
- Alterar PR-07, su sobre Agent v2, el parser legado seguro o su frontera inerte/no privilegiada. PR-07 permanece sin efectos de producto privilegiados.
- Implementar login, admisión, autenticación, transporte, facturación, tarifas, venta de tiempo ni cálculos comerciales adicionales.
- Definir ahora mecanismos físicos concretos de cierre, borrado de perfiles, cierre de sesión Windows, mensajes finales, configuración persistida, migraciones o protocolos.
- Declarar AVAILABLE por timeout, por ausencia de telemetría o porque se cerró una sola aplicación; la reutilización requiere la evidencia de seguridad que se apruebe.

## Recorrido de producto esperado

| Momento | Comportamiento requerido |
| --- | --- |
| Antes del fin normal | Las advertencias configurables informan el vencimiento. Cuando está habilitada, la advertencia final, conceptualmente a un minuto, pregunta si queda trabajo por realizar. |
| Respuesta a gracia | Solo una respuesta afirmativa antes del vencimiento puede habilitar la única gracia. NO o falta de respuesta antes de `00:00` significan que no hay gracia. |
| `00:00` normal con gracia aceptada | Inicia la gracia no facturable configurada y se mantiene escritorio completo; no se ejecuta limpieza mientras el acceso de gracia siga activo. |
| `00:00` normal sin gracia | Termina el acceso y comienza la frontera segura: bloqueo antes de limpieza. |
| Fin temprano o vencimiento de gracia | Termina el acceso y entra en la misma frontera segura, sin segunda gracia. |
| Cualquier terminación normal autorizada | Converge en bloqueo → cierre controlado → timeout de política y fuerza elegible si procede → limpieza/verificación → AVAILABLE solo si es seguro. |
| Fallo o interrupción | La estación queda bloqueada, no AVAILABLE, registra el resultado mínimo y entra al procedimiento de recuperación/contención que se apruebe. |

## Áreas afectadas

| Área | Impacto esperado |
| --- | --- |
| Usuario PC — estado de sesión, temporizador y UI | Advertencias, pregunta de gracia, reloj diferenciado, finalización temprana, bloqueo y estados de contención sin divulgar información sensible. |
| Usuario PC — control Windows y aplicaciones | Orquestación de bloqueo previo, cierre controlado, timeout, terminación elegible, protección de procesos y verificación de reutilización. |
| Usuario PC — navegador, perfil y almacenamiento local | Identificación y eliminación/verificación de residuos de sesión y recuperación segura ante limpieza interrumpida. |
| Dinamizador — política y operación | Observación de resultados de terminación y definición pendiente, únicamente, de la política de advertencia/gracia cuando ordena una terminación remota. |
| Auditoría, soporte y reportería futura | Eventos minimizados; ocupación definida como tiempo normal más gracia efectiva, y latencia de limpieza/disponibilidad potencialmente observable por separado; sin contenido sensible ni definición comercial. |
| Pruebas y operación Windows | Spike obligatorio en equipo Windows real —prerrequisito de planificación autorizado—, escenarios de interrupción y evidencia de que AVAILABLE solo aparece tras limpieza comprobada. |

## Guardrails e invariantes

1. La estación se bloquea antes del cierre lento o de la limpieza; ningún paso posterior puede dejar acceso interactivo expuesto.
2. AVAILABLE/bienvenida es un estado de reutilización segura, no una pantalla decorativa ni un estado inferido por temporizador.
3. Una gracia es excepcional, única, no facturable en esta propuesta y solo nace de una respuesta afirmativa hecha antes del vencimiento normal; NO o falta de respuesta antes de `00:00` no equivalen a consentimiento.
4. La gracia comienza en el `00:00` del tiempo normal y no prolonga ni reescribe el tiempo normal ya registrado.
5. Las causas de terminación convergen en el límite seguro, pero no reciben una gracia implícita por converger.
6. El cierre controlado precede a la fuerza; la fuerza respeta el timeout de política y la lista de procesos protegidos. Nunca se usará «kill-all».
7. La privacidad se demuestra sobre residuos accesibles, no solo sobre procesos cerrados; un resultado no verificable es un fallo para AVAILABLE.
8. La trazabilidad se minimiza: IDs, tiempos, causa, transición y resultado sí; contenido de usuario y secretos no.
9. Una interrupción o ambigüedad durante limpieza conserva bloqueo y exige recuperación/contención explícita; no se borra evidencia ni se reabre acceso por conveniencia.
10. Se aplica `KEEP > EXTEND > REFACTOR > REPLACE > ADD`; no se sustituye la base de bloqueo, recuperación o transporte ya verificada sin evidencia y aprobación posteriores.

## Riesgos y tradeoffs

| Riesgo | Impacto | Contención propuesta |
| --- | --- | --- |
| AVAILABLE antes de limpiar o verificar perfiles | Exposición de datos a la siguiente persona. | Fail-closed; bloquear y mantener fuera de AVAILABLE hasta evidencia aprobada. |
| Cierre forzado indiscriminado | Daño a procesos críticos, soporte o accesibilidad. | Cierre controlado primero, timeout de política y lista explícita de procesos elegibles/protegidos; prohibir kill-all. |
| Limpieza excesiva de perfiles | Pérdida de datos locales o comportamiento Windows inestable. | Spike real que compare alternativas y política institucional explícita sobre archivos locales. |
| Gracia ambigua o repetible | Uso no autorizado, confusión y registros inconsistentes. | Una única gracia, solicitud previa, respuesta positiva explícita y tiempos separados. |
| Reinicio durante limpieza | Residuo de privacidad o estado de estación desconocido. | Mantener bloqueada, persistir trazabilidad mínima y diseñar recuperación/conciliación específica sin modificar PR-06. |
| Evidencia operativa insuficiente en Windows | Un mecanismo aparente puede fallar con perfiles, navegadores o procesos reales. | Spike obligatorio con combinaciones y criterios observables antes de adoptar una estrategia final. |

El tradeoff principal es equilibrar una salida rápida de la estación con la certeza de privacidad y seguridad de reutilización. Esta propuesta prefiere contener una estación más tiempo antes que exponer un residuo o declarar AVAILABLE sin prueba.

## Rollback y contención

- La habilitación futura debe ser reversible por política o estación, conservando el comportamiento seguro de bloqueo y sin eliminar auditoría ni evidencia de limpieza.
- Si la estrategia nueva falla, la estación se mantiene bloqueada y se deriva al procedimiento operativo autorizado; no vuelve a AVAILABLE como rollback automático.
- El rollback no modifica retroactivamente los tiempos normal/gracia/ocupación ya registrados ni inventa cálculos comerciales.
- Los cambios futuros deben ser aditivos y compatibles con la línea base verificada hasta PR-16; la reversión no altera PR-06 ni PR-07.
- La elección final entre cierre controlado, limpieza de perfil y logoff/reset de Windows se apoya en el spike y debe incluir una ruta de reversión y recuperación no destructiva.

## Criterios de éxito

1. Toda finalización de acceso bloquea la estación antes de iniciar cierre o limpieza y nunca muestra AVAILABLE sin verificación de reutilización segura.
2. Las advertencias previas son configurables y, cuando se habilita la advertencia final conceptualmente situada a un minuto, esta pregunta si queda trabajo por realizar; el copy técnico definitivo no se fija aquí.
3. Solo una respuesta afirmativa antes del vencimiento habilita una única gracia, que inicia en el `00:00` normal con escritorio completo; NO o falta de respuesta antes de `00:00` no producen gracia.
4. El fin temprano y el vencimiento de gracia convergen en el mismo cierre seguro, y ninguna otra causa normal obtiene gracia inventada; la política de advertencia/gracia ante orden remota del Dinamizador queda abierta.
5. La ocupación de estación equivale exactamente a la duración normal más la duración efectiva de gracia; la latencia de limpieza/disponibilidad no se incluye y podrá observarse por separado, sin definir tarifas ni cálculos comerciales.
6. La secuencia de cierre intenta primero una salida controlada, respeta el timeout de política para fuerza elegible, nunca ejecuta kill-all y preserva procesos protegidos.
7. La verificación cubre residuos de navegador, perfiles y datos accesibles, no solamente el estado de procesos, y falla de forma cerrada cuando es incompleta o ambigua.
8. La auditoría permite reconstruir causa, secuencia y resultado con datos minimizados, sin contenido de usuario ni secretos.
9. El diseño posterior contiene escenarios de corte de energía, reinicio, fallos parciales y recuperación de limpieza, sin alterar el contrato de recuperación de PR-06.
10. El spike obligatorio y ya autorizado en Windows real compara cierre controlado, limpieza de navegador/perfil, logoff/reset de perfil y combinaciones, y aporta evidencia para escoger una estrategia segura y reversible.

## OPEN REQUIREMENTS — bloquean solo decisiones definitivas e implementación dependiente

La especificación funcional, el diseño y la planificación están autorizados ahora. Los siguientes puntos no se decidirán por inferencia en diseño o código: cada uno bloquea únicamente su decisión técnica definitiva, la implementación productiva correspondiente y las unidades de trabajo que dependan de ella. No bloquean el spike Windows, que ya es un prerrequisito de planificación autorizado.

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

## Siguiente paso recomendado

Avanzar ahora con la especificación funcional, el diseño y la planificación de tareas no dependientes, manteniendo la ubicación posterior a PR-16 sin asignar un número de PR. Ejecutar el spike Windows real como prerrequisito de planificación autorizado y resolver cada OPEN REQUIREMENT antes de su decisión técnica definitiva o implementación dependiente. Se mantienen intactos los contratos cerrados de PR-06 y PR-07 y la línea base validada hasta PR-16.
