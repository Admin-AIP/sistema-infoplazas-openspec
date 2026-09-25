# Especificación de Gestión de Metas

## Propósito

Definir el comportamiento funcional de Gestión de Metas con la Plataforma Central/Web como autoridad administrativa exclusiva, consulta estrictamente de solo lectura (`READ-ONLY`) para el Dinamizador dentro de su propia Infoplaza y separación entre datos operativos, indicadores, metas, evaluaciones y resultados oficiales.

## Alcance

Esta especificación define requisitos funcionales y asuntos abiertos. No autoriza diseño técnico, implementación, cambios de producto ni modificación de documentos históricos.

## Requirements

### Requirement: Autoridad administrativa exclusiva de Central/Web

La Plataforma Central/Web MUST ser la única autoridad administrativa de Gestión de Metas, limitada en cada operación a roles centrales autorizados por una matriz exacta de permisos que permanece abierta. Según corresponda al flujo finalmente aprobado, las operaciones reservadas a Central/Web MUST incluir crear y configurar ciclos; crear, editar y versionar metas; asignar metas; retirar, cancelar o suspender asignaciones; configurar indicadores, fuentes, pesos, consolidación, fechas y ventanas; administrar evidencia; validar resultados o evidencia; gestionar prórrogas; cerrar ciclos; congelar resultados oficiales; efectuar correcciones auditadas posteriores al cierre; producir reportes y rankings; producir agregados provinciales, regionales, de región y nacionales; y ejercer las demás funciones administrativas ya definidas para este dominio. Esta reserva de autoridad MUST NOT resolver por inferencia qué rol central concreto ejecuta cada operación.

#### Scenario: Operación administrativa autorizada

- GIVEN una operación administrativa de Gestión de Metas
- AND un actor autenticado de Central/Web
- WHEN la matriz final de permisos autorice al rol para esa operación
- THEN el sistema MUST permitir que la operación continúe dentro de su alcance autorizado
- AND MUST registrar la acción administrativa conforme a los requisitos de auditoría

#### Scenario: Rol central sin permiso confirmado

- GIVEN una operación administrativa de Gestión de Metas
- AND un actor de Central/Web cuyo permiso exacto no está confirmado
- WHEN el actor intenta ejecutar la operación
- THEN el sistema MUST denegarla
- AND MUST NOT inferir autoridad a partir de la mera pertenencia a Central/Web

### Requirement: Dinamizador estrictamente de solo lectura

El Dinamizador MUST limitarse a consulta y MUST NOT crear, editar, eliminar, configurar, asignar, reasignar, retirar, cancelar, suspender, aceptar, solicitar, aplicar, cargar, modificar, validar, aprobar, rechazar, cambiar ni administrar metas, ciclos, indicadores, fuentes, pesos, fórmulas, resultados, estados, modalidades de consolidación, parámetros de evaluación, rankings, cierres, versiones, configuración o contenido de evidencia, ni configuración de prórrogas. También MUST NOT ejecutar esas acciones sobre dependencias administrativas equivalentes del dominio. Las rutas, API o IPC de lectura MUST carecer de capacidad de mutación y MUST NOT escalar una consulta a escritura. La experiencia de consulta MUST NOT mostrar controles que sugieran capacidades de escritura inexistentes.

#### Scenario: Intento directo de mutación por un Dinamizador

- GIVEN un Dinamizador autenticado
- WHEN intenta mutar cualquier recurso de Gestión de Metas por interfaz, enlace directo, identificador elaborado, API o IPC
- THEN el sistema MUST denegar la operación
- AND MUST NOT alterar datos ni estados del dominio

#### Scenario: Consulta sin controles engañosos

- GIVEN un Dinamizador autorizado para consultar Gestión de Metas
- WHEN accede a la vista permitida
- THEN el sistema MUST presentar una experiencia de solo lectura
- AND MUST NOT presentar controles de creación, edición, aceptación, carga, validación, solicitud, aprobación o administración

#### Scenario: Ruta de lectura usada como vía de escritura

- GIVEN una ruta, API o IPC destinada a consulta
- WHEN una solicitud intenta provocar una mutación directa o indirecta
- THEN el sistema MUST fallar de forma cerrada
- AND MUST conservar sin cambios los recursos consultados

### Requirement: Aislamiento por Infoplaza propia

Toda consulta de metas realizada por un Dinamizador MUST autorizar en backend o servicio el concepto `AUTHORIZED_INFOPLAZA == TARGET_INFOPLAZA` y MUST fallar de forma cerrada ante ausencia, error, discrepancia o ambigüedad. El Dinamizador MUST NOT acceder a metas privadas o asignadas de otro centro, resultados internos detallados, evidencia, historial operativo de metas ni datos administrativos de otra Infoplaza mediante identificadores elaborados, parámetros de consulta, enlaces directos, exportaciones o desgloses. El filtrado en interfaz MUST NOT considerarse autorización suficiente.

#### Scenario: Consulta de la Infoplaza autorizada

- GIVEN un Dinamizador con una Infoplaza autenticada y autorizada
- AND el objetivo de la consulta corresponde a esa misma Infoplaza
- WHEN el backend o servicio evalúa `AUTHORIZED_INFOPLAZA == TARGET_INFOPLAZA`
- THEN el sistema MUST permitir únicamente la información de lectura aprobada para esa Infoplaza

#### Scenario: Intento de consulta cruzada

- GIVEN un Dinamizador autenticado para una Infoplaza
- WHEN solicita información cuyo objetivo pertenece a otra Infoplaza mediante cualquier identificador, consulta, enlace, exportación o desglose
- THEN el sistema MUST denegar la solicitud de forma cerrada
- AND MUST NOT revelar la existencia, contenido ni detalle administrativo del recurso ajeno

#### Scenario: Ámbito no verificable

- GIVEN una consulta cuyo ámbito autorizado u objetivo no puede verificarse sin ambigüedad
- WHEN el backend o servicio intenta autorizarla
- THEN el sistema MUST denegar el acceso
- AND MUST NOT sustituir el objetivo por datos de otra Infoplaza

### Requirement: Vista conceptual Metas de mi Infoplaza

El Dinamizador MUST disponer únicamente de la vista conceptual y de solo lectura `Metas de mi Infoplaza`. Cuando cada dato esté autorizado y disponible, la vista MAY mostrar ciclo activo; metas asignadas; descripción; indicador; período; objetivo o meta; valor real; cumplimiento mensual; resultado consolidado; exceso separado; estado de meta; estado de validación; fecha límite; prórroga concedida; historial mensual; historial de ciclo; información o documentos oficiales; condición provisional u oficial; y notificaciones informativas. La disponibilidad MAY limitar el conjunto mostrado y MUST respetar minimización de datos.

#### Scenario: Consulta autorizada con información disponible

- GIVEN un Dinamizador autorizado para su propia Infoplaza
- AND existe información aprobada y disponible de una meta asignada
- WHEN abre `Metas de mi Infoplaza`
- THEN el sistema MAY mostrar los datos enumerados que correspondan
- AND MUST mantenerlos en modo de solo lectura
- AND MUST distinguir su condición provisional u oficial

#### Scenario: Información no autorizada o no disponible

- GIVEN un Dinamizador en `Metas de mi Infoplaza`
- WHEN un dato no está autorizado o no está disponible
- THEN el sistema MUST NOT exponer ese dato
- AND MUST NOT reemplazarlo por información sensible, administrativa o perteneciente a otro centro

### Requirement: Cadena desacoplada desde datos operativos

Los módulos operativos MUST continuar registrando sus datos normales de atención, usuarios, actividades, capacitación, servicios, sesiones y demás datos ya definidos. Un indicador autorizado MAY consumir automáticamente esos datos sin que dicho consumo constituya edición de metas. Cuando la fuente operativa ya exista, Gestión de Metas SHOULD evitar una segunda captura manual del mismo dato. La relación conceptual MUST conservar la cadena `dato operativo -> indicador -> meta -> vista de solo lectura de la Infoplaza propia`.

#### Scenario: Consumo automático de una fuente existente

- GIVEN un módulo operativo que ya registra un dato autorizado
- AND un indicador configurado para consumir esa fuente
- WHEN el dato alimenta la evaluación de una meta
- THEN el sistema MUST tratarlo como consumo del indicador y no como edición de la meta
- AND SHOULD evitar solicitar la duplicación manual del mismo dato

#### Scenario: Operación cotidiana independiente

- GIVEN que Gestión de Metas está disponible o no disponible
- WHEN un actor registra información en un módulo operativo dentro de su función normal
- THEN el módulo MUST conservar su responsabilidad operativa
- AND MUST NOT convertir ese registro en una capacidad administrativa de metas para el Dinamizador

### Requirement: Evaluación mensual independiente y límite de cumplimiento

Cada meta activa MUST tener una evaluación independiente por cada mes completo incluido en el ciclo. El sistema MUST preservar por separado el valor real, el cumplimiento mensual oficial limitado a un máximo de 100 % y el exceso. El exceso MUST NOT incrementar el peso de la meta ni MUST deshacer empates de manera automática. El sistema MUST NOT declarar un cumplimiento mensual oficial superior a 100 %.

#### Scenario: Resultado por encima del objetivo

- GIVEN una meta mensual con objetivo 100 y valor real 150
- WHEN se determina el cumplimiento mensual oficial
- THEN el sistema MUST preservar el valor real 150
- AND MUST registrar el cumplimiento mensual oficial como máximo en 100 %
- AND MUST preservar el exceso por separado como `+50`
- AND MUST NOT declarar un cumplimiento oficial de 150 %

#### Scenario: Exceso frente a pesos y empates

- GIVEN una evaluación que conserva exceso separado
- WHEN se calcula una ponderación o se comparan resultados empatados
- THEN el exceso MUST NOT aumentar el peso
- AND MUST NOT romper el empate por defecto

### Requirement: Modalidad de consolidación explícita

Cada meta MUST seleccionar explícitamente exactamente una modalidad inicial entre `PROMEDIO_MENSUAL`, `ACUMULATIVO_CICLO` y `EVENTO_VENTANA`. Las metas acumulativas y por evento de ventana MUST conservar evaluaciones mensuales independientes. Las fórmulas exactas de cada modalidad, combinación y tratamiento de datos MUST permanecer abiertas hasta aprobación explícita.

#### Scenario: Configuración de modalidad

- GIVEN una meta que será configurada para un ciclo
- WHEN un rol central debidamente autorizado define su consolidación
- THEN el sistema MUST exigir exactamente una de las tres modalidades iniciales aprobadas
- AND MUST NOT inferir una modalidad adicional

#### Scenario: Evaluación mensual en modalidad no promedio

- GIVEN una meta configurada como `ACUMULATIVO_CICLO` o `EVENTO_VENTANA`
- WHEN transcurre un mes completo del ciclo durante el cual la meta está activa
- THEN el sistema MUST conservar una evaluación independiente para ese mes
- AND MUST mantenerla disponible para el historial mensual

### Requirement: Provisionalidad, validación, cierre y empates

Todos los resultados y rankings durante un ciclo abierto MUST ser provisionales. Un resultado oficial y un ranking oficial MUST existir únicamente después del cierre autorizado del ciclo. Cuando una validación requerida esté en estado `PENDIENTE`, el sistema MUST bloquear el cierre oficial y MUST NOT contabilizar esa validación como cumplimiento oficial. Los empates MUST permanecer como empates y el exceso MUST NOT utilizarse como criterio predeterminado de desempate.

#### Scenario: Consulta durante un ciclo abierto

- GIVEN un ciclo que todavía no ha sido cerrado
- WHEN se consulta un resultado o ranking
- THEN el sistema MUST identificarlo como provisional
- AND MUST NOT presentarlo como oficial

#### Scenario: Validación requerida pendiente

- GIVEN al menos una validación requerida en estado `PENDIENTE`
- WHEN un rol central intenta cerrar oficialmente el ciclo
- THEN el sistema MUST bloquear el cierre
- AND MUST NOT contar la validación pendiente como cumplimiento oficial

#### Scenario: Resultado empatado

- GIVEN dos resultados empatados conforme a las reglas aprobadas
- WHEN se presenta su posición comparativa
- THEN el sistema MUST conservar el empate
- AND MUST NOT usar el exceso como desempate automático

### Requirement: Asignación en lugar de exoneración o no aplicabilidad

El conjunto inicial del dominio MUST NOT incluir los estados o conceptos de estado `EXONERADA` ni `NO_APLICA`. Cuando una meta no corresponda a una Infoplaza, el sistema MUST representarlo mediante la ausencia de asignación y MUST NOT asignarla para luego marcarla como exonerada o no aplicable.

#### Scenario: Meta que no corresponde a una Infoplaza

- GIVEN una meta que no aplica a una Infoplaza determinada
- WHEN se establecen las asignaciones del ciclo
- THEN el sistema MUST dejar la meta sin asignar a esa Infoplaza
- AND MUST NOT utilizar `EXONERADA` ni `NO_APLICA`

### Requirement: Estados separados y conjuntos iniciales exactos

El estado de meta y el estado de validación MUST ser dimensiones separadas. El conjunto inicial de estados de meta MUST ser exactamente `PENDIENTE`, `EN_PROGRESO`, `CUMPLIDA`, `INCUMPLIDA`, `PRORROGA_CONCEDIDA`, `SUSPENDIDA` y `CANCELADA`. El conjunto inicial de estados de validación MUST ser exactamente `NO_REQUIERE`, `PENDIENTE`, `APROBADA` y `RECHAZADA`. El sistema MUST NOT mezclar ambos conjuntos ni añadir estados iniciales no aprobados.

#### Scenario: Representación independiente de estados

- GIVEN una meta con estado propio y una evaluación sujeta a validación
- WHEN se consulta su situación
- THEN el sistema MUST representar por separado el estado de meta y el estado de validación
- AND cada valor MUST pertenecer a su conjunto inicial exacto

### Requirement: Ciclos de meses completos y semáforo configurable

Todo ciclo MUST comenzar el primer día de un mes y MUST terminar el último día de un mes, sin meses parciales. Los pesos y umbrales de semáforo MUST ser configurables por autoridad central autorizada y MUST NOT estar codificados como valores fijos. Los estados conceptuales iniciales de semáforo MUST ser exactamente `EN_RITMO`, `EN_RIESGO`, `RETRASADO` y `CUMPLIDA`; sus umbrales exactos permanecen abiertos.

#### Scenario: Período válido de ciclo

- GIVEN la creación o configuración de un ciclo
- WHEN se establecen sus fechas de inicio y fin
- THEN la fecha de inicio MUST ser el primer día de un mes
- AND la fecha de fin MUST ser el último día de un mes
- AND el ciclo MUST NOT contener meses parciales

#### Scenario: Configuración sin valores rígidos

- GIVEN pesos y umbrales aplicables a un ciclo
- WHEN un rol central autorizado los configura
- THEN el sistema MUST aceptar valores conforme a las reglas finalmente aprobadas
- AND MUST NOT imponer valores hardcodeados como regla funcional

### Requirement: Separación y fuente explícita de indicadores

Cada indicador MUST permanecer conceptualmente separado de las metas que lo utilicen y MUST declarar una fuente explícita entre automática, manual controlada o basada en evidencia y validación. El mapeo final de fuente para cada indicador MUST permanecer abierto hasta aprobación explícita.

#### Scenario: Meta vinculada a un indicador

- GIVEN una meta que utiliza una medición
- WHEN se configura su indicador
- THEN el sistema MUST mantener identidades conceptuales separadas para la meta y el indicador
- AND el indicador MUST declarar exactamente una categoría de fuente aprobada

#### Scenario: Mapeo de fuente no aprobado

- GIVEN un indicador cuyo origen final todavía no ha sido aprobado
- WHEN se documenta o configura su dependencia
- THEN el sistema MUST mantener el mapeo como asunto abierto
- AND MUST NOT seleccionar una fuente por defecto inferida

### Requirement: Congelamiento, correcciones e historia

El cierre autorizado MUST congelar los resultados oficiales. Toda corrección posterior al cierre MUST registrar como mínimo quién la realizó, cuándo, el motivo, el valor anterior y el valor nuevo. El sistema MUST preservar el historial mensual y el historial consolidado del ciclo. Los cambios de meta o versión MUST preservar trazabilidad y MUST conservar el concepto `REQUIERE_NUEVA_ACEPTACION` cuando la regla final aprobada determine que aplica, sin resolver por inferencia el actor ni el flujo de aceptación.

#### Scenario: Cierre de ciclo sin pendientes bloqueantes

- GIVEN un ciclo elegible para cierre conforme a las reglas aprobadas
- AND no existe una validación requerida pendiente
- WHEN un rol central autorizado cierra el ciclo
- THEN el sistema MUST congelar sus resultados como oficiales
- AND MUST preservar el historial mensual y consolidado

#### Scenario: Corrección posterior al cierre

- GIVEN un resultado oficial congelado
- WHEN se autoriza una corrección posterior al cierre
- THEN el registro de auditoría MUST incluir quién, cuándo, motivo, valor anterior y valor nuevo
- AND el sistema MUST preservar la historia previa y la trazabilidad del cambio

#### Scenario: Cambio versionado con posible nueva aceptación

- GIVEN un cambio de meta o versión
- WHEN la regla final aprobada determine que requiere nueva aceptación
- THEN el sistema MUST representar `REQUIERE_NUEVA_ACEPTACION`
- AND MUST NOT atribuir silenciosamente la aceptación al Dinamizador ni a otro actor no aprobado

### Requirement: Corrección prospectiva de supuestos anteriores

Para Gestión de Metas, esta especificación MUST corregir prospectivamente cualquier supuesto anterior que atribuya escritura al Dinamizador. Las operaciones claramente administrativas MUST quedar reservadas a Central/Web, sujetas a la matriz final. Los propietarios y flujos no resueltos MUST permanecer abiertos; en particular, el sistema MUST NOT asignar silenciosamente a un nuevo actor la aceptación digital, la carga o validación de evidencia ni la solicitud o aprobación de prórrogas. Los documentos históricos MUST permanecer sin modificación por este cambio.

#### Scenario: Supuesto histórico de escritura local

- GIVEN un supuesto anterior que permita al Dinamizador mutar Gestión de Metas
- WHEN se evalúa comportamiento futuro bajo esta especificación
- THEN el sistema MUST aplicar el modelo de Dinamizador estrictamente de solo lectura
- AND MUST NOT restaurar la capacidad de escritura histórica

#### Scenario: Propietario de flujo todavía abierto

- GIVEN un flujo de aceptación, evidencia o prórroga cuyo actor exacto no está aprobado
- WHEN se especifica su comportamiento futuro
- THEN el propietario y el flujo MUST permanecer abiertos
- AND MUST NOT asignarse por inferencia al Dinamizador, a Usuario PC ni a un rol central concreto

### Requirement: Exclusión de Usuario PC

Usuario PC MUST permanecer fuera de Gestión de Metas y MUST NOT conocer, mostrar, administrar, aceptar, clasificar, ordenar, evidenciar ni presentar rankings de metas. Usuario PC MAY producir datos operativos desacoplados que indicadores autorizados consuman indirectamente. Esta especificación MUST NOT autorizar cambios de producto en Usuario PC.

#### Scenario: Uso normal de Usuario PC

- GIVEN una persona que utiliza Usuario PC
- WHEN opera sus funciones normales
- THEN Usuario PC MUST NOT presentar ni administrar Gestión de Metas
- AND MUST NOT ofrecer aceptación, evidencia o ranking de metas

#### Scenario: Dato operativo desacoplado

- GIVEN un dato operativo producido por Usuario PC fuera de Gestión de Metas
- WHEN un indicador autorizado lo consume indirectamente
- THEN el consumo MAY contribuir a la medición del indicador
- AND MUST NOT crear conocimiento, interfaz ni autoridad de metas en Usuario PC

### Requirement: Seguridad, privacidad y auditoría

Gestión de Metas MUST aplicar privilegio mínimo, autorización en backend o servicio, aislamiento entre centros y minimización de datos sensibles. Las consultas del Dinamizador MUST carecer de capacidad de mutación y ninguna ruta, API o IPC de consulta MUST permitir escalamiento a escritura. Las acciones administrativas de Central/Web MUST ser auditables. El sistema MUST NOT exponer entre centros metas, resultados detallados, evidencia, historia operativa ni datos administrativos.

#### Scenario: Solicitud con privilegios insuficientes

- GIVEN un actor autenticado sin el privilegio requerido
- WHEN solicita lectura o administración de Gestión de Metas
- THEN el backend o servicio MUST denegar la operación
- AND MUST minimizar cualquier información incluida en la respuesta

#### Scenario: Acción administrativa central

- GIVEN un rol central autorizado
- WHEN realiza una acción administrativa de Gestión de Metas
- THEN el sistema MUST generar trazabilidad auditable de la acción
- AND MUST limitar la información tratada a la necesaria para su propósito autorizado

## OPEN REQUIREMENTS

Los asuntos siguientes permanecen expresamente sin resolver. El sistema MUST NOT seleccionar valores predeterminados, actores, fórmulas ni consecuencias por inferencia para cerrar estas decisiones:

1. La fórmula exacta para combinar resultados ponderados y no ponderados, incluida la consolidación de cada modalidad.
2. Los validadores autorizados por cada tipo de evidencia y el flujo exacto de validación, aprobación o rechazo.
3. La autoridad exacta para conceder, solicitar, aplicar, modificar o revocar prórrogas y sus efectos funcionales.
4. La matriz exacta de roles y permisos de Central/Web, incluida la segregación de funciones administrativas.
5. Las reglas de reconocimiento y sus relaciones con resultados, empates y rankings.
6. Los umbrales exactos de `EN_RITMO`, `EN_RIESGO`, `RETRASADO` y `CUMPLIDA`.
7. El mapeo final de la fuente de cada indicador y las reglas de calidad, conciliación y disponibilidad del dato.
8. La retención, integridad, acceso, evidencia mínima y tratamiento de evidencia parcial o tardía.
9. Los formatos finales, destinatarios, periodicidad, descarga, accesibilidad, retención y nivel de detalle de reportes y documentos.
10. El actor y flujo de aceptación digital, las condiciones y consecuencias de `REQUIERE_NUEVA_ACEPTACION`, la falta de aceptación y la compatibilidad con el Dinamizador estrictamente de solo lectura.
11. Las fórmulas exactas, precisión, redondeo, tratamiento de datos faltantes y sus consecuencias para consolidación, cumplimiento y ranking.
12. La necesidad, alcance, seguridad, revocación, retención y frescura de una caché offline de solo lectura.
13. La visibilidad y audiencia de publicación de rankings provisionales y oficiales.
14. Los efectos de correcciones posteriores al cierre sobre resultados oficiales, reportes, rankings, notificaciones, versiones, republicación y comunicación.
15. La autoridad, requisitos, límites y efectos exactos de retirar, suspender o cancelar asignaciones sobre evaluaciones, historia, cierre y resultados.
16. El propietario y flujo de carga de evidencia, sin atribuirlo por defecto al Dinamizador ni a otro actor.
17. Las consecuencias funcionales de impedir al Dinamizador solicitar prórrogas o aceptar digitalmente, incluidos los canales alternativos, si fueran aprobados.
18. Las reglas exactas de agregación provincial, regional, de región y nacional, junto con su privacidad y audiencia.
19. La auditoría de acceso de lectura, la minimización adicional y el comportamiento ante permisos cambiantes o datos incompletos.
20. Cualquier otra consecuencia dependiente de una decisión no resuelta por la propuesta aprobada.

## Lista final de consistencia y alcance

- [x] La especificación reserva la administración exclusivamente a Central/Web y mantiene abierta su matriz exacta de permisos.
- [x] La especificación limita al Dinamizador a lectura y excluye toda mutación, escalamiento y control engañoso.
- [x] La especificación exige autorización cerrada por `AUTHORIZED_INFOPLAZA == TARGET_INFOPLAZA` en backend o servicio.
- [x] La especificación restringe `Metas de mi Infoplaza` a información autorizada, disponible y de solo lectura.
- [x] La especificación conserva la cadena desacoplada de dato operativo, indicador, meta y consulta de la Infoplaza propia.
- [x] La especificación exige evaluación mensual, límite oficial de 100 %, valor real y exceso separado sin peso ni desempate automático.
- [x] La especificación limita las modalidades iniciales a `PROMEDIO_MENSUAL`, `ACUMULATIVO_CICLO` y `EVENTO_VENTANA` sin inventar fórmulas.
- [x] La especificación distingue resultados provisionales y oficiales, bloquea el cierre por validación requerida pendiente y conserva empates.
- [x] La especificación excluye `EXONERADA` y `NO_APLICA` y representa la no aplicabilidad mediante ausencia de asignación.
- [x] La especificación mantiene separados y exactos los conjuntos iniciales de estados de meta y validación.
- [x] La especificación exige ciclos de meses completos y pesos y umbrales configurables, no hardcodeados.
- [x] La especificación separa indicadores y metas y mantiene abierta la fuente final de cada indicador.
- [x] La especificación congela resultados al cierre, audita correcciones y conserva historia, versiones y aceptación condicional abierta.
- [x] La especificación corrige prospectivamente la escritura atribuida al Dinamizador sin modificar documentos históricos.
- [x] La especificación mantiene Usuario PC fuera de Gestión de Metas y solo admite datos operativos desacoplados sin autorizar cambios de producto.
- [x] La especificación incluye privilegio mínimo, autorización de servicio, aislamiento, auditoría y minimización de datos.
- [x] La especificación mantiene explícitamente abiertos permisos, fórmulas, validación, evidencia, prórrogas, aceptación, reconocimiento, reportes, caché, publicación y correcciones.
- [x] El alcance se limita a esta especificación funcional y no incluye diseño, tareas, implementación, ramas, worktrees, staging, commits, pushes ni PR.
- [x] El alcance no incluye cambios PR-08 ni cambios en `Soft_Dinamizador/`, `Soft_Usuario_PC/` o cualquier producto.
