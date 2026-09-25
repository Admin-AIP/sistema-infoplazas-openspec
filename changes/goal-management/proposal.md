# Propuesta: Gestión de Metas con autoridad central y consulta acotada por Infoplaza

- **Cambio:** `goal-management`
- **Estado:** propuesta de requisitos y especificación funcional únicamente. Autoriza elaborar especificaciones posteriores, pero **no autoriza implementación de producto, código, diseño técnico, tareas, ramas, worktrees, staging, commits, pushes ni PRs**.
- **Autoridad de producto confirmada:** la Plataforma Central/Web es la autoridad administrativa exclusiva. El Dinamizador solo consulta, en modo lectura (`READ-ONLY`), las metas de su propia Infoplaza autenticada.
- **Precedencia:** para la Gestión de Metas, esta propuesta sustituye las suposiciones anteriores que sean contrarias a estas reglas, sin reescribir, editar ni invalidar documentos históricos. Los flujos anteriores de escritura atribuidos al Dinamizador quedan corregidos prospectivamente conforme a esta propuesta.

## Intención y problema

La Gestión de Metas requiere una fuente administrativa única, resultados comparables y una delimitación estricta entre la administración central y la consulta local. Atribuir escritura o decisiones de validación al Dinamizador crea riesgo de alteración no autorizada, resultados inconsistentes, exposición entre Infoplazas y confusión sobre quién responde por el ciclo y sus resultados oficiales.

El resultado esperado es que los roles autorizados de la Plataforma Central/Web administren el ciclo completo de metas con trazabilidad, mientras cada Dinamizador pueda comprender el desempeño aprobado de su propia Infoplaza sin poder alterar metas, evidencias, validaciones, cálculos, rankings ni estados.

## Resultado normativo

1. La Plataforma Central/Web es la única autoridad administrativa para ciclos, metas, versiones, asignaciones, indicadores, fuentes, pesos, consolidación, ventanas, evidencia, validación, prórrogas, cierre, congelamiento oficial, correcciones auditadas, reportes, rankings y agregados. La matriz exacta de permisos entre sus roles autorizados permanece abierta.
2. El Dinamizador accede exclusivamente a la vista conceptual **`Metas de mi Infoplaza`**, limitada a la Infoplaza de su identidad autenticada y estrictamente de solo lectura.
3. Toda autorización de lectura de metas, resultados detallados, evidencias, historial y datos administrativos se verifica en backend/servicio bajo el concepto `AUTHORIZED_INFOPLAZA == TARGET_INFOPLAZA`. Ante ausencia, error o ambigüedad, la respuesta falla cerrada. El filtrado de interfaz nunca constituye autorización suficiente.
4. Usuario PC queda fuera de Gestión de Metas: no muestra, administra, acepta, clasifica, ordena ni evidencia metas. Puede producir datos operativos desacoplados que indicadores autorizados consuman indirectamente.
5. La captura operativa sigue en sus módulos funcionales normales —usuarios, servicios, actividades, capacitación, sesiones y equivalentes—. El consumo automático de esos datos por un indicador no es edición de una meta ni autoriza doble digitación manual.

## Alcance propuesto

### Incluye

1. Definir el dominio central de ciclos y metas, incluidas versiones, asignación por Infoplaza, indicadores, fuente de indicador, pesos, períodos, ventanas, consolidación, evidencias, validaciones, extensiones, cierre, congelamiento y correcciones posteriores al cierre.
2. Mantener indicadores como entidades separadas de las metas y declarar explícitamente su fuente: **automática**, **manual controlada** o **basada en evidencia/validación**. El mapeo definitivo de fuentes operativas permanece abierto.
3. Evaluar cada meta activa de manera independiente en cada mes del ciclo, incluso cuando la modalidad principal sea acumulativa o por evento de ventana.
4. Soportar explícitamente las modalidades `PROMEDIO_MENSUAL`, `ACUMULATIVO_CICLO` y `EVENTO_VENTANA`. Las evaluaciones mensuales subsisten en las tres modalidades; la fórmula exacta de combinación queda abierta.
5. Mantener para cada evaluación mensual el porcentaje real, el cumplimiento oficial mensual limitado a 100 %, y el exceso por separado. El exceso no suma peso ni deshace empates.
6. Distinguir el resultado consolidado en curso y los rankings provisionales de los resultados y rankings oficiales. Durante el ciclo todo resultado o ranking es provisional; solo el cierre habilita la condición oficial.
7. Mantener separadamente los estados de meta `PENDIENTE`, `EN_PROGRESO`, `CUMPLIDA`, `INCUMPLIDA`, `PRORROGA_CONCEDIDA`, `SUSPENDIDA` y `CANCELADA`, y los estados de validación `NO_REQUIERE`, `PENDIENTE`, `APROBADA` y `RECHAZADA`. No se incorporan `EXONERADA` ni `NO_APLICA`.
8. Exigir que una validación pendiente requerida bloquee el cierre y no se contabilice como cumplimiento oficial. Las reglas de quienes validan permanecen abiertas.
9. Definir ciclos que comienzan el primer día y terminan el último día de meses completos, sin meses parciales; soportar pesos configurables y umbrales configurables de semáforo. `EN_RITMO`, `EN_RIESGO`, `RETRASADO` y `CUMPLIDA` son categorías conceptuales, sujetas a umbrales finales aprobados.
10. Cerrar el ciclo mediante autoridad central y congelar los resultados oficiales. Toda corrección posterior registra, como mínimo, quién, cuándo, motivo, valor anterior y valor nuevo; se conserva la historia mensual y de ciclo, así como la trazabilidad de versiones y la regla `REQUIERE_NUEVA_ACEPTACION` cuando aplique la regla final aprobada.
11. Presentar al Dinamizador, solo cuando estén disponibles y autorizados para su propia Infoplaza: ciclo activo, metas asignadas, descripción, indicador, período, meta objetivo, valor real, porcentaje mensual limitado, resultado consolidado, exceso separado, estados de meta y validación, fecha límite, prórroga concedida, historial mensual y de ciclo, información/documentos oficiales, condición provisional/oficial y notificaciones informativas.
12. Aplicar minimización de datos, privilegio mínimo, auditoría de administración central y prohibición de que una ruta, API o IPC de lectura escale a mutación. No se exponen a un Dinamizador datos de otra Infoplaza, ni evidencia, historial, resultado detallado o información administrativa fuera de su ámbito.

### No incluye

- Implementar interfaces, servicios, esquemas, integraciones, permisos técnicos, migraciones, reportes, ranking, aceptación, sincronización, caché offline, pruebas o cualquier cambio de producto.
- Otorgar al Dinamizador creación, edición, eliminación, configuración, asignación/reasignación, cancelación/suspensión, aceptación, carga/modificación de evidencia, solicitud/aplicación de prórrogas, validación, aprobación/rechazo, o cambio de estados, resultados, fórmulas, rankings o versiones.
- Hacer visible Gestión de Metas en Usuario PC, ni otorgarle administración, aceptación, ranking, evidencia o una dependencia acoplada al dominio de metas.
- Duplicar la captura de resultados en Gestión de Metas cuando los módulos operativos autorizados ya generan los datos de origen.
- Resolver por inferencia los requisitos abiertos de permisos, fórmulas, aceptación, validación, reconocimiento, retención, publicación, correcciones o presentación final.

## Recorrido de producto esperado

| Momento | Comportamiento requerido |
| --- | --- |
| Administración de ciclo y metas | Un rol central autorizado administra el ciclo, versiones, asignaciones, indicadores y reglas aprobadas; ningún actor Dinamizador puede mutar esos datos. |
| Operación cotidiana | Los módulos operativos registran su información normal; un indicador autorizado puede consumirla de forma automática sin crear una segunda entrada de meta. |
| Consulta local | El Dinamizador autenticado abre `Metas de mi Infoplaza` y recibe únicamente información aprobada de su propia Infoplaza, sin controles que sugieran acciones administrativas. |
| Evaluación mensual | Cada meta activa tiene evaluación independiente mensual; se preservan real, cumplimiento limitado a 100 % y exceso separado. |
| Validación y cierre | Una validación requerida pendiente bloquea el cierre y no es cumplimiento oficial; al cerrar, Central congela resultados oficiales. |
| Corrección posterior | Una corrección autorizada posterior al cierre mantiene resultado e historial trazables y deja auditoría de actor, tiempo, motivo, anterior y nuevo. |
| Solicitud indebida o cruzada | Toda lectura fuera de `AUTHORIZED_INFOPLAZA == TARGET_INFOPLAZA`, o toda mutación desde Dinamizador/ruta de lectura, se deniega sin exponer información. |

## Áreas afectadas

| Área | Impacto esperado |
| --- | --- |
| Plataforma Central/Web | Única superficie de administración y autoridad de ciclos, metas, indicadores, validación, consolidación, cierre, correcciones, reportes, rankings y agregados. |
| Dominio de metas e indicadores | Separación entre meta, indicador, fuente, evaluación mensual, consolidación, estados, versiones, evidencia, validación e historial. |
| Seguridad y autorización | Control backend/servicio por Infoplaza autenticada, privilegio mínimo, denegación cerrada, prevención de escalamiento de lectura a escritura y auditoría central. |
| Dinamizador | Consulta de solo lectura de `Metas de mi Infoplaza`; sin acciones engañosas, datos cruzados ni capacidades administrativas. |
| Módulos operativos | Conservan la entrada funcional de datos; sus datos pueden alimentar indicadores autorizados de forma desacoplada. |
| Usuario PC | Sin impacto funcional directo ni superficie de Gestión de Metas; solo posible generación indirecta de datos operativos desacoplados. |
| Reportería y auditoría | Resultados provisionales/oficiales, empates preservados, exceso separado, historial, congelamiento y correcciones auditables. |

## Guardrails e invariantes

1. La autoridad administrativa de metas reside exclusivamente en Central/Web; el Dinamizador no posee una excepción de escritura.
2. La frontera de Infoplaza se aplica en el servidor o servicio, no solo en navegación, filtros o componentes de interfaz, y falla cerrada ante cualquier discrepancia.
3. Una ruta de consulta, API o IPC no puede convertirse en vía de mutación ni revelar detalle administrativo, evidencia o historial fuera de la Infoplaza autorizada.
4. Las metas y los indicadores son conceptos separados; la fuente de un indicador debe declararse explícitamente y el consumo de datos operativos no duplica su captura.
5. Cada meta activa se evalúa mensualmente; las modalidades no eliminan el historial mensual ni autorizan meses parciales.
6. El cumplimiento mensual oficial no supera 100 %, aunque el valor real y el exceso se preserven por separado. El exceso no altera pesos ni rompe empates; los empates permanecen empates.
7. Los estados de meta y validación son dimensiones independientes y solo usan los valores aprobados en esta propuesta.
8. Los resultados y rankings intraciclo son provisionales. El cierre autorizado es la condición para congelar y publicar el resultado oficial, y una validación pendiente requerida impide ese cierre.
9. La corrección posterior al cierre conserva trazabilidad inmutable de versión, actor, tiempo, motivo, valor previo y valor nuevo; no borra el historial mensual o de ciclo.
10. La información mostrada al Dinamizador se minimiza a lo aprobado y disponible para su propia Infoplaza; no se inventan acciones administrativas, reglas de aceptación ni datos sensibles adicionales.

## Riesgos y tradeoffs

| Riesgo | Impacto | Contención propuesta |
| --- | --- | --- |
| Filtrado solo en UI o confianza en un identificador enviado por cliente | Fuga cruzada entre Infoplazas y acceso indebido a evidencias o resultados. | Autorizar en backend/servicio contra el ámbito autenticado y fallar cerrada. |
| Mantener escritura histórica en Dinamizador | Alteración no autorizada y ausencia de responsabilidad central. | Retirar prospectivamente toda mutación de ese actor y concentrarla en roles centrales autorizados. |
| Fórmula o ponderación inferida | Rankings, reconocimiento y resultados discutibles. | Mantener abiertas fórmula, redondeo y reglas de reconocimiento hasta aprobación explícita. |
| Cierre con validación incompleta | Declaración prematura de cumplimiento oficial. | Bloquear cierre y condición oficial mientras exista validación requerida pendiente. |
| Exceso integrado a ponderación o desempate | Incentivos y comparaciones injustas. | Limitar el porcentaje mensual oficial a 100 %, conservar exceso separado y preservar empates. |
| Corrección posterior sin evidencia de cambio | Pérdida de confianza, auditoría y capacidad de conciliación. | Auditar actor, tiempo, motivo, anterior, nuevo, versión e historial. |
| Lectura local demasiado detallada | Exposición de información administrativa o sensible no necesaria. | Minimización, lista explícita de campos aprobados y ausencia de datos fuera de la Infoplaza autenticada. |

El tradeoff principal es privilegiar control central, comparabilidad y privacidad sobre la conveniencia de editar localmente. Esta propuesta prefiere denegar una operación ambigua y mantener la consulta local limitada antes que permitir cambios o exposición sin autoridad verificable.

## Rollback y contención

- Cualquier implementación futura deberá poder deshabilitar la exposición de consulta para una Infoplaza o actor sin conceder escritura local ni eliminar auditoría, historiales, versiones o resultados ya congelados.
- Ante fallo de autorización, discrepancia de ámbito, fuente no confiable o estado no verificable, el sistema niega acceso o evita declarar resultados oficiales; no presenta datos alternativos de otra Infoplaza.
- Una reversión futura no restituye las capacidades de escritura del Dinamizador que esta propuesta retira ni convierte datos provisionales en oficiales.
- Las correcciones posteriores al cierre son ajustes auditados, no borrado o reemplazo silencioso de resultados históricos.
- La reversión de integraciones con módulos operativos debe conservar el origen y la trazabilidad disponibles, sin forzar captura manual duplicada de metas.

## Criterios de éxito

1. Solo roles autorizados de la Plataforma Central/Web pueden administrar cualquier elemento del ciclo de vida de Gestión de Metas.
2. El Dinamizador solo puede consultar `Metas de mi Infoplaza` para su Infoplaza autenticada y no encuentra acciones ni rutas que permitan mutar metas o sus dependencias.
3. Toda solicitud de datos de metas aplica `AUTHORIZED_INFOPLAZA == TARGET_INFOPLAZA` en backend/servicio y falla cerrada sin filtración cruzada de metas, resultados detallados, evidencias, historial o datos administrativos.
4. Los indicadores se distinguen de las metas, tienen fuente explícita y pueden consumir datos operativos autorizados sin duplicar su entrada manual.
5. Cada meta activa conserva evaluación mensual en `PROMEDIO_MENSUAL`, `ACUMULATIVO_CICLO` y `EVENTO_VENTANA`, dentro de ciclos de meses completos.
6. El resultado mensual oficial está limitado a 100 %, mientras valor real y exceso se conservan separados; el exceso no agrega peso ni rompe empates.
7. Metas y validaciones mantienen estados independientes aprobados, no existen `EXONERADA` ni `NO_APLICA`, y una validación requerida pendiente bloquea cierre y oficialidad.
8. Los resultados/rankings son provisionales durante el ciclo, el cierre central los congela como oficiales y toda corrección posterior queda auditada con valores anterior/nuevo y motivo.
9. La vista local presenta únicamente los campos aprobados y disponibles, con información provisional/oficial clara y sin inducir a acciones administrativas.
10. Usuario PC no adquiere interfaz, permisos, aceptación, evidencia, ranking ni conciencia del dominio de metas; su contribución posible permanece desacoplada como dato operativo.

## OPEN REQUIREMENTS — bloquean decisiones definitivas e implementación dependiente

Los siguientes asuntos no se resolverán silenciosamente por especificación derivada, diseño o código. Cada uno bloquea solo su decisión final y la implementación que dependa de ella; esta propuesta no autoriza implementación alguna.

1. **Matriz de permisos Central:** roles concretos y segregación de funciones para administrar, validar, aprobar, cerrar, corregir, publicar, reportar y configurar.
2. **Fórmulas y redondeo:** combinación ponderada/no ponderada, consolidación por modalidad, precisión, redondeo, tratamiento de datos faltantes y consecuencias en ranking.
3. **Validación y evidencia:** validadores habilitados, flujo de aprobación/rechazo, evidencia mínima, retención, acceso, integridad y tratamiento de evidencia parcial o tardía.
4. **Prórrogas, suspensión y cancelación:** autoridad, requisitos, límites, efecto sobre evaluaciones, estado, cierre, historial y resultados provisionales/oficiales.
5. **Aceptación digital:** actor, flujo, alcance, condición de `REQUIERE_NUEVA_ACEPTACION`, consecuencias de no aceptar y compatibilidad con el Dinamizador estrictamente de solo lectura.
6. **Semáforo, reconocimiento y ranking:** umbrales exactos, reglas de reconocimiento, publicación, audiencia, tratamiento de empates y comportamiento de rankings provisionales.
7. **Fuentes de indicadores:** mapeo final a módulos operativos, calidad y disponibilidad del dato, controles de captura manual controlada y conciliación de fuentes automáticas.
8. **Correcciones posteriores:** autoridad, efectos exactos sobre resultados oficiales, reportes, rankings, notificaciones, versiones, republicación y comunicación a partes afectadas.
9. **Reportes y documentos:** formatos, destinatarios, periodicidad, descarga, retención, accesibilidad y nivel de detalle permitido por Infoplaza.
10. **Lectura resiliente y privacidad:** necesidad y reglas de caché de consulta offline, revocación, retención, minimización adicional, auditoría de acceso y comportamiento con datos incompletos o permisos cambiantes.

## Siguiente paso recomendado

El siguiente paso es convertir estos requisitos confirmados y abiertos en una especificación funcional revisable, sin diseñar ni implementar mecanismos técnicos. Las decisiones abiertas deben contar con aprobación explícita de producto, operación, seguridad y autoridad central antes de autorizar cualquier trabajo de producto dependiente. Esta propuesta no modifica documentos históricos, ni autoriza un PR o una implementación.
