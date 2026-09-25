# Plan de trabajo futuro: cierre seguro, gracia y limpieza

**Estado:** planificación solamente. Este documento no autoriza implementación de producto, cambios de contratos, operaciones Git ni asignación de números de PR nuevos. Todas las unidades de producto descritas abajo son trabajo futuro y requieren autorización posterior de apply, además de sus dependencias y gates aplicables; el spike de evidencia es la única actividad no productiva planificada. La línea base entre repositorios debe estar verificada hasta el PR-16 existente.

## Review Workload Forecast

| Field | Value |
| ------- | ------- |
| Estimated changed lines | TBD after spike and decision gates |
| 400-line budget risk | High |
| Chained PRs recommended | Yes |
| Suggested split | Spike de evidencia → dominio local y trazabilidad mínima → UI conductual neutral → coordinador/adaptadores → recuperación, integración remota, reportería y rollout |
| Delivery strategy | ask-on-risk |
| Chain strategy | pending |

Decision needed before apply: Yes
Chained PRs recommended: Yes
Chain strategy: pending
400-line budget risk: High

## Límites y disciplina futura

- Cada futura unidad autónoma debe mantenerse en **≤400 líneas cambiadas** (adiciones + eliminaciones), incluidos tests y documentación; si no cabe, se subdivide antes de apply sin comprimir código, pruebas ni documentación.
- TDD estricto en cada futura unidad implementable: **RED → GREEN → TRIANGULATE → REFACTOR**. Esta planificación no ejecuta ni afirma evidencia de esas etapas.
- Preservar exactamente: `ocupación = tiempo normal + gracia efectiva`. La latencia de limpieza/disponibilidad se mide por separado y no amplía ocupación, tiempo normal ni gracia efectiva; no se añaden tarifas ni cálculos comerciales.
- Conservar los contratos de PR-05, PR-06 y PR-07. PR-06 mantiene su recuperación de arranque y temporizador durable; PR-07 permanece limitado al sobre Agent v2 y parser legado seguro separado, sin tipos productivos ni efectos privilegiados.
- Reutilizar `Soft_Usuario_PC/agente-infoplaza/internal/lockdown` y sus rutas MAINTENANCE/ADMIN y RECOVERY/WATCHDOG; no crear un segundo bloqueo ni redefinir la recuperación de PR-06.

## Unidad futura 1 — spike Windows real A/B/C/D

**Estado:** evidencia de planificación autorizada; no es implementación de producto. **Inicio:** línea base PR-16 verificada y estación Windows de laboratorio representativa disponible. **Final:** informe comparativo revisable y reversión experimental comprobada. **Dependencias precisas:** únicamente línea base PR-16 verificada y disponibilidad del laboratorio Windows; no depende de ningún OPEN REQUIREMENT. **Rollback:** restaurar la configuración/perfil experimental documentado y confirmar estación bloqueada/fuera de `AVAILABLE` hasta recuperación segura.

- [ ] Preparar en `Soft_Usuario_PC/agente-infoplaza` y en el laboratorio Windows un protocolo reproducible y matriz sintética para A: cierre controlado; B: cierre y limpieza de navegador/perfil; C: logoff Windows controlado o reset de perfil; y D: combinaciones con orden explícito, registrando versiones de Windows, navegador, aplicaciones y política experimental. <!-- sdd-owner: implementation -->
- [ ] Ejecutar A/B/C/D en Windows real con navegador autenticado sintético, descargas/caché/cookies/historial/autocompletado sintéticos, documentos guardado y no guardado sintéticos, aplicación cooperativa/no respondiente, proceso protegido/no clasificable, fallo de perfil/almacenamiento, desconexión, crash/reinicio y repetición; evidenciar bloqueo previo a trabajo lento, privacidad residual, estabilidad, recuperación, compatibilidad institucional, autostart de agentes, UX, duración, trabajo no guardado y retorno seguro o rechazo de `AVAILABLE`. <!-- sdd-owner: implementation -->
- [ ] Publicar bajo `openspec/changes/session-end-grace-cleanup/` un informe por categorías/códigos con resultados conocidos/desconocidos, mediciones separadas de ocupación y latencia, prueba de no-`kill-all` y rollback no destructivo de cada variante, sin perfiles, URLs, credenciales, títulos de ventana ni contenido personal. <!-- sdd-owner: implementation -->

## Puertas de decisión obligatorias

Ninguna unidad futura dependiente puede iniciar implementación productiva sin su resolución explícita, evidencia del spike cuando corresponda y autorización posterior de apply. Los gates no bloquean comportamiento local confirmado que no dependa de ellos.

| Gate | OPEN REQUIREMENT que debe resolverse | Desbloquea solamente |
| --- | --- | --- |
| G1 | Política Dinamizador ante terminación remota. | Rama de terminación remota e integración Dinamizador que aplique esa política. |
| G2 | Estrategia final de limpieza basada en spike A/B/C/D. | Adaptadores de estrategia, verificador y rollout de dicha estrategia. |
| G3 | Timeout, autoridad y conducta del cierre controlado. | Política/adaptador de cierre selectivo. |
| G4 | Lista, autoridad y excepciones de procesos protegidos/elegibles. | Política/adaptador de cierre selectivo. |
| G5 | Navegadores, datos y límites del mecanismo de navegador/perfil. | Limpieza de navegador/perfil y su evidencia de privacidad. |
| G6 | Política para documentos no guardados tras gracia. | UX y política de cierre de aplicaciones para esos documentos. |
| G7 | Estado durable, conciliación y atención de limpieza interrumpida. | Recuperación posterior al fin. |
| G8 | Copy, accesibilidad e idiomas finales de UI. | Texto final, decisiones de accesibilidad/idiomas y liberación de ese copy. |
| G9 | Destinatarios, retención, presentación y reporte adicional/final. | Reportería final y proyecciones adicionales para esos destinatarios. |
| G10 | Política institucional de archivos locales de usuario. | Limpieza de perfil/archivos y verificación correspondiente. |

## Unidades futuras posteriores (sin números de PR)

Cada unidad tiene frontera de rollback: revertir solo su cambio compatible; una limpieza pendiente o ambigua permanece bloqueada y fuera de `AVAILABLE` hasta conciliación segura.

### Dominio local, tiempos y gracia confirmada — FUTURA, no autorizada ahora

**Inicio:** base PR-16 verificada y autorización posterior de apply. **Final:** decisiones durables e idempotentes para advertencia local, respuesta previa, única gracia, fin normal/voluntario/anticipado y tiempos separados. **Dependencias:** base PR-16; no depende de G1, pues la causa remota queda excluida; no depende de G8, pues no fija copy/UI. **Verificación futura:** Go RED → GREEN → TRIANGULATE → REFACTOR. **Rollback:** revertir la extensión compatible sin reabrir acceso ni alterar registros históricos.

- [ ] RED: en los objetivos de descubrimiento `Soft_Usuario_PC/agente-infoplaza/internal/session/` y persistencia de la base PR-16, añadir pruebas de respuesta local afirmativa/negativa/ausente/tardía, una sola gracia, replay/concurrencia, fin normal, fin voluntario y fin anticipado, excluyendo explícitamente terminación remota. <!-- sdd-owner: implementation -->
- [ ] GREEN: en `Soft_Usuario_PC/agente-infoplaza/internal/session/` y su persistencia/base PR-16, extender la decisión durable e idempotente para esos casos locales, preservando tiempo normal, gracia concedida, gracia efectiva, ocupación y latencia separada, sin modificar PR-06. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar `Soft_Usuario_PC/agente-infoplaza/internal/session/**/*_test.go` con límites `00:00`, reintentos, reinicios y fin temprano para demostrar que ocupación es solo normal más gracia efectiva y que la latencia no se incorpora. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: acotar interfaces y nombres en `Soft_Usuario_PC/agente-infoplaza/internal/session/` sin cambiar semántica ni introducir una rama de causa remota. <!-- sdd-owner: implementation -->

### Trazabilidad mínima local y ocupación — FUTURA, no autorizada ahora

**Inicio:** dominio local futuro disponible y autorización posterior de apply. **Final:** eventos mínimos no sensibles y métricas temporales separadas para el camino local confirmado. **Dependencias:** dominio local y sinks descubiertos en `Soft_Usuario_PC/agente-infoplaza/internal/logging/`; no depende de G9. G9 solo condiciona destinatarios, retención, presentación y reportería adicional/final. **Verificación futura:** allowlist y cálculo temporal con TDD estricto. **Rollback:** detener esta proyección compatible sin borrar auditoría existente.

- [ ] RED: en `Soft_Usuario_PC/agente-infoplaza/internal/logging/`, auditoría/outbox y tests descubiertos, definir pruebas de allowlist para equivalentes funcionales de fin, gracia, bloqueo, cleanup, verificación, contención y recuperación, junto con normal, gracia efectiva, ocupación y latencia separados, rechazando contenido, URLs, archivos, credenciales, formularios y payloads crudos. <!-- sdd-owner: implementation -->
- [ ] GREEN: extender los objetivos `Soft_Usuario_PC/agente-infoplaza/internal/logging/`, auditoría/outbox y modelo de sesión PR-16 con los eventos mínimos y datos temporales confirmados, calculando ocupación exactamente como normal más gracia efectiva y latencia aparte. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar los tests de logging/auditoría descubiertos bajo `Soft_Usuario_PC/agente-infoplaza/internal/` para fin normal, gracia parcial y limpieza prolongada, comprobando que los datos permitidos se emiten y los sensibles no. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: consolidar la construcción allowlist de eventos en los objetivos descubiertos de `Soft_Usuario_PC/agente-infoplaza/internal/logging/` sin añadir destinatarios, retención, vistas ni reporte final dependientes de G9. <!-- sdd-owner: implementation -->

### UI conductual local neutral — FUTURA, no autorizada ahora

**Inicio:** dominio local futuro disponible y autorización posterior de apply. **Final:** la UI recoge respuesta explícita, distingue reloj normal/gracia y permite fin voluntario sin fijar copy final. **Dependencias:** dominio local y bindings Wails de la base PR-16; no depende de G1 porque no implementa terminación remota; no depende de G8 para pruebas de comportamiento con marcadores neutrales. La producción del texto final, accesibilidad e idiomas queda bloqueada por G8. **Verificación futura:** frontend y bindings RED → GREEN → TRIANGULATE → REFACTOR. **Rollback:** retirar la proyección compatible; el fin local conserva bloqueo fail-closed.

- [ ] RED: en `Soft_Usuario_PC/agente-infoplaza/frontend/src/components/TimerTab.test.tsx`, `SessionEndedOverlay.test.tsx` y bindings descubiertos desde `app_lifecycle.go`, añadir pruebas neutrales al copy para advertencia local habilitada/deshabilitada, respuesta explícita, reloj normal/gracia distinguible y fin voluntario, sin decidir textos, idiomas o detalles de accesibilidad. <!-- sdd-owner: implementation -->
- [ ] GREEN: en `Soft_Usuario_PC/agente-infoplaza/frontend/src/components/TimerTab.tsx`, `SessionEndedOverlay.tsx`, `LockScreen.tsx` y bindings Wails descubiertos desde `app_lifecycle.go`, proyectar los estados y acciones locales confirmados sin conceder gracia en UI ni dejar escritorio accesible tras el fin. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar `Soft_Usuario_PC/agente-infoplaza/frontend/src/components/*test.tsx` con respuesta tardía, re-render/retry, gracia anticipadamente finalizada y ausencia de segunda gracia, usando selectores y marcadores neutrales al copy. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: aislar la proyección de estado local en los componentes y bindings anteriores sin introducir copy de producción, política remota ni una segunda autoridad temporal. <!-- sdd-owner: implementation -->

### Rama remota e integración Dinamizador — BLOCKED (G1; G9 solo para reportería adicional)

**Inicio:** G1 resuelto, unidades locales aplicables disponibles y autorización posterior de apply. **Final:** Dinamizador emite/observa una terminación remota conforme a la política aprobada, sin alterar el camino local confirmado. **Dependencias:** G1, dominio local, contratos verificados hasta PR-16; G9 únicamente si se agrega reportería final/adicional. **Verificación futura:** TDD de servicio único, idempotencia y proyecciones minimizadas; no cambios al wire de PR-07. **Rollback:** cesar la extensión remota sin ejecutar limpieza remota paso a paso ni modificar PR-07.

- [ ] RED: en los objetivos de descubrimiento `Soft_Dinamizador/apps/desktop/` y el vínculo existente de `Soft_Usuario_PC/agente-infoplaza/`, añadir pruebas para la precedencia remota aprobada por G1, idempotencia de intención y proyección no sensible, sin decidir reportería de G9. <!-- sdd-owner: implementation -->
- [ ] GREEN: extender aditivamente los objetivos anteriores mediante el servicio único de sesión para emitir/observar intención, etapa y resultado minimizados conforme a G1, sin orquestación Windows remota, sesión paralela ni ampliación del sobre Agent v2/parser legado de PR-07. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar los tests de integración descubiertos en `Soft_Dinamizador/apps/desktop/test/` y `Soft_Usuario_PC/agente-infoplaza/**/*_test.go` con reenvío, desconexión y conflicto de intención, verificando que no se crea gracia ni ejecución paralela. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: mantener adaptadores LAN/IPC finos hacia el servicio único descubierto en `Soft_Dinamizador/apps/desktop/`, sin incorporar retención, destinatarios, vistas ni reporte final de G9. <!-- sdd-owner: implementation -->

### Coordinador seguro y frontera de lock — BLOCKED (spike completo y G2)

**Inicio:** estrategia aprobada y dominio disponible. **Final:** una única frontera coordinada permite cleanup solo tras lock confirmado. **Dependencias:** spike, G2, dominio de fin, `Soft_Usuario_PC/agente-infoplaza/internal/lockdown/` y `app_lock.go` de base PR-16. **Verificación futura:** TDD de coordinación y fallo de lock. **Rollback:** deshabilitar ruta nueva sin desbloquear pendientes.

- [ ] RED: crear pruebas en el paquete de orquestación por descubrir bajo `Soft_Usuario_PC/agente-infoplaza/internal/` que demuestren fin efectivo → bloqueo confirmado → trabajo lento → evidencia → gate `AVAILABLE`, y que un fallo, duplicado o incertidumbre de lock impide cleanup y disponibilidad. <!-- sdd-owner: implementation -->
- [ ] GREEN: añadir el coordinador idempotente detrás de interfaces en el paquete descubierto, reutilizando `internal/lockdown` y manteniendo fail-closed. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar `Soft_Usuario_PC/agente-infoplaza/internal/**/*_test.go` con causas convergentes, fallo de bloqueo y evidencia cruzada/vencida para probar serialización y rechazo de `AVAILABLE`. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: acotar wiring de `Soft_Usuario_PC/agente-infoplaza/app_lock.go` y el paquete coordinador sin crear un segundo bloqueo ni cambiar MAINTENANCE/ADMIN o RECOVERY/WATCHDOG. <!-- sdd-owner: implementation -->

### Política/adaptadores de cierre de procesos — BLOCKED (G3, G4 y G6; spike completo)

**Inicio:** política aprobada y coordinador disponible. **Final:** cierre selectivo devuelve resultado clasificable y fail-closed. **Dependencias:** G3, G4, G6, spike y coordinador. **Verificación futura:** TDD de política/adaptador. **Rollback:** deshabilitar adaptador y contener estación.

- [ ] RED: en objetivos por descubrir bajo `Soft_Usuario_PC/agente-infoplaza/internal/`, añadir pruebas de cierre controlado antes de fuerza, timeout aprobado, solo elegibles, protegidos/no clasificables nunca forzados, documentos no guardados según G6 y ausencia de `kill-all`. <!-- sdd-owner: implementation -->
- [ ] GREEN: implementar detrás de interfaces del coordinador la clasificación minimizada y adaptador Windows de solicitud controlada, espera, reevaluación y fuerza selectiva conforme a G3/G4/G6. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar pruebas de política/adaptador con proceso protegido, no clasificable, no respondiente y resultados desconocidos, verificando atención y no `AVAILABLE`. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: separar política de adaptador en los objetivos descubiertos bajo `Soft_Usuario_PC/agente-infoplaza/internal/` sin capturar títulos o contenido de ventanas. <!-- sdd-owner: implementation -->

### Limpieza de privacidad y verificación/AVAILABLE — BLOCKED (G2, G5 y G10; spike completo)

**Inicio:** estrategia y políticas aprobadas, coordinador disponible. **Final:** evidencia completa es la única ruta a `AVAILABLE`. **Dependencias:** G2, G5, G10, spike y coordinador. **Verificación futura:** TDD de adaptador/verificador. **Rollback:** deshabilitar estrategia sin publicar disponibilidad.

- [ ] RED: en objetivos de descubrimiento `Soft_Usuario_PC/agente-infoplaza/internal/`, añadir pruebas que rechacen proceso cerrado con residuos, evidencia incompleta, timeout, silencio y evidencia de otra sesión/estación. <!-- sdd-owner: implementation -->
- [ ] GREEN: implementar los puertos de estrategia de navegador/perfil, agregación de evidencia minimizada, verificador independiente y gate de disponibilidad conforme a G2/G5/G10. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar los tests del verificador con residuos de categorías aprobadas, fallo de adaptador, evidencia vencida/contradictoria y éxito completo de la misma sesión/estación. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: aislar la estrategia elegida detrás de los puertos descubiertos sin permitir que un comando ejecutado, timeout o ausencia de telemetría implique `AVAILABLE`. <!-- sdd-owner: implementation -->

### Recuperación de cleanup interrumpido — BLOCKED (G7; spike completo)

**Inicio:** política durable aprobada y coordinador disponible. **Final:** interrupciones se detectan y permanecen contenidas hasta conciliación. **Dependencias:** G7, spike, coordinador y estado durable base PR-16. **Verificación futura:** TDD de reinicio/crash en cada frontera. **Rollback:** binario compatible conserva pendientes y requiere atención.

- [ ] RED: en un módulo futuro de recuperación bajo `Soft_Usuario_PC/agente-infoplaza/internal/`, añadir pruebas de crash/reinicio, pérdida de energía, almacenamiento y red en cada frontera, comprobando que ninguna ruta devuelve `AVAILABLE` sin nueva verificación. <!-- sdd-owner: implementation -->
- [ ] GREEN: añadir detección, contención, conciliación/reintento o atención de terminación/limpieza incompleta conforme a G7, integrándose después de recuperación de arranque PR-06 sin redefinirla. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar `Soft_Usuario_PC/agente-infoplaza/internal/**/*_test.go` con estados pendientes, evidencia ambigua y reanudación idempotente para demostrar que no se duplica gracia ni orquestación. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: separar la recuperación de limpieza del contrato existente de PR-06 y conservar los pendientes compatibles con rollback. <!-- sdd-owner: implementation -->

### Reportería final y proyecciones adicionales — BLOCKED (G9)

**Inicio:** G9 resuelto, trazabilidad mínima local disponible y autorización posterior de apply. **Final:** destinatarios, retención, presentación y reporte adicional/final se ajustan a la decisión aprobada. **Dependencias:** G9 y trazabilidad mínima; los eventos mínimos, ocupación y latencia separados no esperan G9. **Verificación futura:** TDD de proyecciones y retención aprobadas. **Rollback:** detener solo las proyecciones nuevas sin borrar auditoría existente.

- [ ] RED: en objetivos de descubrimiento `Soft_Usuario_PC/agente-infoplaza/internal/logging/`, auditoría/outbox y `Soft_Dinamizador/apps/desktop/`, añadir pruebas para destinatarios, retención, presentación y datos adicionales aprobados por G9, manteniendo la exclusión de contenido sensible. <!-- sdd-owner: implementation -->
- [ ] GREEN: implementar únicamente las proyecciones, almacenamiento de reporte y vistas aprobadas por G9 en los objetivos descubiertos, reutilizando los eventos mínimos ya emitidos. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar tests de los objetivos anteriores con destinatarios autorizados/no autorizados, límites de retención y presentaciones vacías o incompletas, sin modificar los cálculos de ocupación o latencia. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: centralizar controles de acceso y retención aprobados en los objetivos de reportería descubiertos sin incorporar payloads crudos ni redefinir la auditoría local. <!-- sdd-owner: implementation -->

### UI final de copy, accesibilidad e idiomas — BLOCKED (G8)

**Inicio:** G8 resuelto y UI conductual local futura disponible. **Final:** texto final de producción y sus decisiones de accesibilidad/idioma implementados conforme a G8. **Dependencias:** G8; no reabre la conducta local ya probada con marcadores neutrales. **Verificación futura:** TDD de copy, accesibilidad e idiomas aprobados. **Rollback:** volver a la proyección neutral compatible sin alterar la gracia ni el bloqueo.

- [ ] RED: en `Soft_Usuario_PC/agente-infoplaza/frontend/src/components/TimerTab.test.tsx`, `SessionEndedOverlay.test.tsx` y `LockScreen` tests descubiertos, añadir pruebas de los textos, semántica accesible e idiomas expresamente aprobados por G8. <!-- sdd-owner: implementation -->
- [ ] GREEN: aplicar el copy, accesibilidad e idiomas aprobados por G8 en `Soft_Usuario_PC/agente-infoplaza/frontend/src/components/TimerTab.tsx`, `SessionEndedOverlay.tsx` y `LockScreen.tsx`. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ampliar pruebas frontend con cada idioma/tecnología asistiva y estados de advertencia, gracia, bloqueo y atención aprobados. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: extraer recursos de texto aprobados en los objetivos frontend descubiertos sin cambiar los contratos conductuales ni revelar detalles sensibles de limpieza. <!-- sdd-owner: implementation -->

### Prueba Windows/E2E, AVAILABLE y rollout — BLOCKED (gates aplicables; spike y unidades previas)

**Inicio:** gates pertinentes resueltos, spike y unidades previas concluidos, y autorización posterior de apply. **Final:** piloto por allowlist demuestra rollout y rollback seguros. **Dependencias:** no presupone que todos los gates bloqueen todos los escenarios: la cobertura local confirmada requiere sus unidades locales; terminación remota requiere G1; estrategia de cleanup requiere G2/G5/G10; recuperación requiere G7; reportería final requiere G9; copy final requiere G8. **Verificación futura:** E2E Windows real, flags, rollback y mixed-version; dividir por escenario si supera 400 líneas. **Rollback:** apagar flags por estación y mantener pendientes contenidas.

- [ ] RED: crear en `Soft_Usuario_PC/agente-infoplaza/frontend/src/**/*.test.tsx`, paquetes Go `*_test.go` y `Soft_Dinamizador/apps/desktop/test/` los escenarios negativos de `AVAILABLE`, compatibilidad y rollout gradual aplicables a cada gate resuelto. <!-- sdd-owner: implementation -->
- [ ] GREEN: incorporar feature flags, allowlist por estación y kill switches en los objetivos de configuración/wiring descubiertos bajo `Soft_Usuario_PC/agente-infoplaza/` y, cuando corresponda, `Soft_Dinamizador/apps/desktop/`, sin desbloquear una estación ambigua. <!-- sdd-owner: implementation -->
- [ ] TRIANGULATE: ejecutar la futura cobertura por escenarios de fin normal/voluntario/gracia, remoto aprobado, procesos/privacidad, interrupción, rollback y mixed-version, manteniendo cada unidad dentro del presupuesto. <!-- sdd-owner: implementation -->
- [ ] REFACTOR: consolidar la configuración de rollout y las rutas de rollback en los objetivos descubiertos sin alterar PR-05/PR-06/PR-07 ni declarar `AVAILABLE` por timeout o silencio. <!-- sdd-owner: implementation -->
