# Diseño conceptual: fin de sesión, gracia única y limpieza segura

- **Cambio:** `session-end-grace-cleanup`
- **Fase:** diseño solamente; no autoriza implementación ni operaciones Git.
- **Alcance principal futuro:** `Soft_Usuario_PC/agente-infoplaza`, con integración acotada en `Soft_Dinamizador` y contratos entre repositorios.
- **Base requerida:** línea verificada del cambio `session-start-login-usuario-pc` completa hasta PR-16.
- **Restricción de secuencia:** este cambio no recibe un número de PR nuevo en este documento.
- **Fuentes normativas:** propuesta aprobada y especificación `usuario-pc-session-end`.

## 1. Decisión arquitectónica

El fin de sesión se diseñará como una **frontera coordinada de revocación, limpieza y prueba**, separada de la pantalla de bienvenida y separada del temporizador visible. Todas las terminaciones normales convergen en un coordinador local de fin de sesión. El coordinador termina la autorización de acceso, exige confirmación de bloqueo antes de iniciar trabajo lento, dirige el cierre y la limpieza mediante puertos sustituibles, reúne evidencia y solo habilita `AVAILABLE` cuando un verificador independiente confirma una estación segura para reutilización.

La arquitectura es deliberadamente neutral respecto del mecanismo final. No decide entre cierre de aplicaciones, limpieza de navegador/perfil, cierre de sesión de Windows, reset de perfil o combinaciones. Esas alternativas se compararán en el spike obligatorio sobre Windows real y la elección seguirá abierta hasta aprobación explícita del OPEN REQUIREMENT correspondiente.

El principio rector es:

```text
fin efectivo del acceso
  → bloqueo confirmado
  → cierre y limpieza bajo política
  → verificación basada en evidencia
  → AVAILABLE solo con prueba suficiente
```

Un fallo, una interrupción, evidencia incompleta o resultado desconocido nunca se convierten en disponibilidad por timeout. La estación queda fuera de `AVAILABLE` y entra en contención, recuperación o atención.

## 2. Contexto técnico y límites de la base

El código visible actual confirma superficies que deberán evolucionar sobre la base verificada hasta PR-16:

- `internal/lockdown` y `app_lock.go` concentran la FSM física, el modo kiosco, hooks, foco y watchdog; no se creará un segundo sistema de bloqueo.
- `internal/commands/dispatcher.go` contiene hoy un contador en memoria y bloquea al vencer; la base hasta PR-16 sustituye esa premisa por sesión y temporizador durables.
- `TimerTab.tsx`, `SessionEndedOverlay.tsx` y `LockScreen.tsx` son superficies presentacionales existentes, pero hoy no representan gracia, limpieza verificada ni atención.
- `internal/logging` dispone de sinks locales; su modelo de mensajes libres no debe recibir datos personales ni evidencia detallada sin una allowlist segura.
- El diseño aprobado de inicio asigna a PR-06 la recuperación de arranque y temporizador durable, y a PR-07 el sobre v2 inerte con separación segura del parser legado.

Este cambio **consume**, pero no redefine, la línea base posterior a PR-16. La implementación futura debe validar primero que esa base está integrada y verificada en cada repositorio. OpenSpec representa dependencias de contrato entre repositorios; no implica ancestría Git compartida.

## 3. Flujo de extremo a extremo

### 3.1 Camino principal

1. La política de sesión proyecta advertencias configurables antes del fin normal.
2. Cuando la advertencia final está habilitada, la UI presenta la pregunta conductual de gracia; la UI no concede la gracia por sí misma.
3. Una respuesta afirmativa válida recibida antes de `00:00` puede preparar la única gracia de la sesión. Una respuesta negativa, ausente o tardía no la concede.
4. En `00:00` normal:
   - con gracia válida preparada, termina el tiempo normal, empieza la gracia y continúa el escritorio completo;
   - sin gracia válida, termina inmediatamente la autorización de acceso.
5. El vencimiento, el fin voluntario, el fin anticipado de gracia y cualquier otra terminación normal autorizada convergen en la misma frontera segura, sin inventar gracia.
6. El coordinador declara conceptualmente la sesión terminada y solicita de inmediato el bloqueo físico. No intercala cierre, borrado, reporte, espera de red ni otra tarea lenta antes de esa solicitud y su confirmación.
7. Solo con bloqueo confirmado se inicia la orquestación de cierre y limpieza.
8. El verificador evalúa la evidencia requerida de revocación, cierre, privacidad y seguridad de reutilización.
9. Solo un resultado verificado y suficiente permite publicar `AVAILABLE` y mostrar bienvenida.

### 3.2 Gracia activa

Durante una gracia válida:

- el usuario conserva **el escritorio completo**, no una vista parcial;
- el tiempo restante de gracia es visible y distinguible del tiempo normal;
- no empieza cierre, limpieza ni verificación de reutilización;
- finalizar anticipadamente consume solo la porción efectivamente transcurrida;
- el vencimiento o fin anticipado cruza la misma frontera segura que cualquier terminación normal;
- ningún retry, reinicio, demora de UI o cambio incidental de conectividad crea otra gracia.

Preservar el escritorio completo es válido **únicamente** mientras la gracia esté activa y autorizada. Una vez terminado el acceso, ninguna UI de “procesando” puede dejar superficies de escritorio utilizables detrás de ella.

## 4. Máquina de estados conceptual

Los nombres siguientes sirven para razonar, probar y documentar. **No prescriben enums, columnas, tablas, estados wire ni nombres de eventos persistidos.**

```text
ACTIVE
  ├─ fin normal + gracia válida preparada ───────────────► GRACE
  └─ fin efectivo sin gracia / fin voluntario / causa ──► SESSION_ENDED

GRACE
  └─ vencimiento o fin anticipado ──────────────────────► SESSION_ENDED

SESSION_ENDED
  └─ revocación física inmediata y confirmada ──────────► LOCKED
       └─ inicio de trabajo lento ──────────────────────► CLEANUP_IN_PROGRESS
            ├─ evidencia completa y verificación OK ───► VERIFIED_READY / AVAILABLE
            └─ fallo, incompleto o desconocido ────────► CLEANUP_FAILED / ATTENTION

SESSION_ENDED o LOCKED
  └─ bloqueo fallido/ambiguo o interrupción ────────────► ATTENTION
```

### 4.1 Semántica de los conceptos

| Concepto | Acceso interactivo | Significado |
| --- | --- | --- |
| `ACTIVE` | Permitido | Sesión dentro del tiempo normal autorizado. |
| `GRACE` | Permitido, escritorio completo | Única gracia válida ya concedida y actualmente activa. |
| `SESSION_ENDED` | No autorizado | Frontera lógica de fin; su primer efecto protector es solicitar bloqueo inmediato. |
| `LOCKED` | Revocado y confirmado | La capacidad física de bloqueo aportó evidencia suficiente para continuar. |
| `CLEANUP_IN_PROGRESS` | Revocado | Cierre, tratamiento de privacidad y verificación se ejecutan con la estación bloqueada. |
| `VERIFIED_READY / AVAILABLE` | Solo bienvenida/admisión siguiente | La evidencia aprobada demostró reutilización segura. |
| `CLEANUP_FAILED / ATTENTION` | No permitido | Hay fallo, incompletitud, ambigüedad o recuperación pendiente. |

`SESSION_ENDED → LOCKED` es una transición protectora inmediata: no se usa `SESSION_ENDED` como una espera visible ni como permiso para ejecutar limpieza lenta. Si el bloqueo no puede confirmarse, el flujo no avanza como si estuviera seguro; aplica la máxima contención disponible, se marca fuera de servicio y se deriva a atención.

### 4.2 Invariantes de transición

1. Solo `ACTIVE` o una `GRACE` válida permiten escritorio completo.
2. No hay transición de `SESSION_ENDED` de vuelta a `ACTIVE` o `GRACE` por retry.
3. No hay transición de `CLEANUP_IN_PROGRESS` a `AVAILABLE` por tiempo transcurrido.
4. Un resultado desconocido equivale a no seguro para el gate de disponibilidad.
5. La bienvenida es una proyección de `AVAILABLE`, nunca su causa.
6. La recuperación puede reanudar o conciliar trabajo, pero debe volver a verificar antes de `AVAILABLE`.

## 5. Límites de responsabilidad

| Componente | Responsabilidad | No debe hacer |
| --- | --- | --- |
| **Sesión, temporizador y política de gracia** | Determinar fin normal, advertencias, validez temporal de la respuesta, inicio/fin de gracia y causa efectiva de terminación; proyectar tiempos desde la sesión durable de la base. | Controlar ventanas/procesos, limpiar perfiles, declarar bloqueo físico o decidir `AVAILABLE`. |
| **UI Usuario PC** | Mostrar advertencias, recoger respuesta explícita, distinguir reloj normal y de gracia, ofrecer fin voluntario, y presentar estados mínimos de bloqueo/atención/bienvenida. | Conceder gracia localmente, extender relojes, iniciar limpieza, inferir disponibilidad o revelar detalles de residuos/fallos sensibles. |
| **Bloqueo de estación** | Revocar acceso interactivo mediante la capacidad física ya verificada y devolver evidencia de éxito, fallo o incertidumbre. | Ejecutar limpieza, ocultar errores, convertir bookkeeping en prueba física o alterar las rutas privilegiadas de mantenimiento/recuperación. |
| **Coordinador de fin de sesión** | Serializar la frontera común, deduplicar la intención lógica, exigir bloqueo antes del trabajo lento, coordinar etapas y mantener el resultado fail-closed. | Implementar detalles Win32, elegir la estrategia final, omitir el verificador o liberar la estación por timeout. |
| **Política de aplicaciones/procesos** | Clasificar aplicaciones administradas, elegibles, protegidas o no clasificables; autorizar cierre controlado y, tras timeout, fuerza selectiva elegible. | Hacer `kill-all`, forzar procesos protegidos/no clasificables o tratar “proceso ausente” como prueba integral de privacidad. |
| **Adaptador de aplicaciones/procesos** | Enumerar objetivos permitidos, solicitar cierre controlado, observar resultado y aplicar solo la fuerza expresamente autorizada por política. | Decidir por sí mismo elegibilidad, cambiar la lista protegida o capturar títulos/contenido de ventanas en logs. |
| **Adaptador de navegador/perfil/privacidad** | Aplicar la estrategia aprobada y producir evidencia minimizada sobre las categorías de residuos cubiertas. | Elegir qué navegadores/archivos se borran, declarar seguridad global o registrar URLs, cuentas, nombres de archivos o contenido. |
| **Verificador** | Evaluar evidencia de bloqueo, terminación, alcance de limpieza y ausencia de residuos accesibles conforme a criterios aprobados. | Confiar solo en “comando ejecutado”, ausencia de telemetría, cierre de una aplicación o timeout. |
| **Auditoría y métricas** | Registrar IDs internos, tiempos, causa, etapa, resultado y códigos no sensibles; derivar ocupación y latencia por separado. | Registrar documentos, archivos, URLs, credenciales, formularios, cuentas, payloads crudos o contenido personal. |
| **Integración Dinamizador** | Originar una terminación autorizada, proyectar avance/resultado no sensible y facilitar atención operativa mediante el vínculo seguro de la base. | Ejecutar limpieza local remotamente paso a paso, asumir éxito por desconexión o resolver la política aún abierta de advertencia/gracia remota. |
| **Recuperación y atención** | Contener una estación no verificable, detectar trabajo incompleto, conciliar/reintentar según política aprobada y exigir nueva verificación. | Reabrir acceso por conveniencia, borrar evidencia, apropiarse del contrato de PR-06 o seleccionar anticipadamente persistencia/reintentos. |

La seguridad local no depende de que Dinamizador permanezca conectado después de emitir una terminación. La pérdida de comunicación puede retrasar su observación, pero no habilita acceso ni `AVAILABLE`.

## 6. Contratos conceptuales entre componentes

Los contratos siguientes describen información y postcondiciones, no esquemas exactos ni nombres de wire/evento.

### 6.1 Intención de terminación

Entrada lógica con identidad estable de sesión, estación, causa autorizada, momento observado y correlación de la intención. Repetir la misma intención no vuelve a terminar, no concede gracia y no inicia orquestaciones paralelas. Una intención conflictiva se contiene y audita; no sobrescribe silenciosamente el hecho de fin.

### 6.2 Decisión de gracia

Representa, conceptualmente:

- si la pregunta estaba habilitada y vigente;
- si hubo respuesta afirmativa explícita antes de `00:00`;
- duración concedida por política;
- si la oportunidad única quedó preparada, iniciada, rechazada, consumida o terminada anticipadamente;
- duración efectivamente usada.

El contrato debe permitir una decisión atómica e idempotente a nivel de dominio, pero **no selecciona almacenamiento, tabla, campo, lease ni protocolo**.

### 6.3 Resultado de bloqueo

Distingue al menos confirmación suficiente, fallo conocido y resultado incierto. Solo la confirmación suficiente abre la etapa lenta. Un ACK de solicitud, un estado interno o una pantalla fullscreen sin postcondición física no bastan por sí solos.

### 6.4 Plan y resultado de cierre

La política entrega clasificación y permisos; el adaptador devuelve resultados por objetivo usando identificadores o clases no sensibles. El plan separa:

1. solicitud de cierre controlado;
2. espera del timeout de política;
3. reevaluación de procesos restantes;
4. cierre forzado únicamente para objetivos administrados y elegibles;
5. fallo/atención si un protegido, no clasificable o no terminable impide demostrar seguridad.

La duración, autoridad y detalles del timeout permanecen abiertos.

### 6.5 Evidencia de limpieza y verificación

El orquestador reúne evidencia de varias fuentes, y el verificador emite una decisión independiente. La evidencia debe responder, sin capturar contenido personal, si:

- la sesión terminó y el acceso fue revocado;
- las operaciones aprobadas se ejecutaron sobre el alcance esperado;
- aplicaciones elegibles y procesos restantes cumplen política;
- navegador, perfil, descargas, temporales, credenciales y sesiones web quedaron tratados y verificados conforme a la estrategia aprobada;
- no existe una categoría requerida sin observar;
- no hay interrupción o ambigüedad pendiente.

El “éxito de comando” es evidencia de ejecución, no necesariamente evidencia de privacidad. La composición exacta de la prueba queda ligada al spike y a la resolución de los OPEN REQUIREMENTS.

### 6.6 Gate de disponibilidad

El gate acepta únicamente un conjunto de evidencia completo, vigente para la misma sesión/estación/ejecución y con todas las comprobaciones críticas satisfactorias. Rechaza:

- evidencia faltante, vencida, contradictoria o de otra ejecución;
- bloqueo no confirmado;
- cierre parcial que comprometa verificación;
- residuos detectados;
- adaptador no soportado cuando su alcance era requerido;
- timeout, desconexión o silencio usados como supuesto de éxito.

No se fija aquí una estructura de mensaje ni un nombre exacto de resultado.

## 7. Gracia única: anti-repetición e idempotencia

La oportunidad de gracia pertenece a la **identidad estable de la sesión**, no al componente de UI, conexión, proceso ni cuenta regresiva en memoria.

Reglas conceptuales:

1. Solo una respuesta afirmativa explícita y anterior al fin normal puede preparar la oportunidad.
2. Preparar, iniciar o consumir la gracia vuelve irreversible el hecho de que esa sesión utilizó su única oportunidad.
3. Una respuesta negativa o el vencimiento sin respuesta cierra la oportunidad; una respuesta tardía no la reabre.
4. Repetir la misma solicitud devuelve el resultado ya conocido sin sumar duración ni reiniciar el comienzo.
5. Dos solicitudes concurrentes se resuelven como una única decisión lógica; como máximo una puede producir gracia.
6. Reinicio, reconexión, replay, re-render de UI o reenvío de Dinamizador no cambian el resultado.
7. El fin anticipado fija la duración efectiva y no deja “saldo” reutilizable.
8. La limpieza nunca convierte una sesión terminada en elegible para otra gracia.

La implementación posterior debe demostrar estas propiedades de forma durable, pero el mecanismo de persistencia y su schema no se deciden en este diseño.

## 8. Semántica temporal y métricas

| Magnitud | Inicio y fin | Uso permitido |
| --- | --- | --- |
| **Tiempo normal** | Desde el inicio efectivo de sesión hasta su `00:00` normal o fin anticipado aplicable. | Duración normal de acceso; conserva su semántica previa y no se reescribe al conceder gracia. |
| **Gracia concedida** | Duración máxima aprobada por política para la única gracia. | Explica el límite autorizado; no equivale a tiempo usado. |
| **Gracia efectiva** | Desde `00:00` normal hasta vencimiento o fin anticipado de gracia. | Tiempo excepcional realmente usado; puede ser menor que la gracia concedida. |
| **Ocupación** | Tiempo normal + gracia efectiva. | Medida exacta de uso interactivo de estación para este cambio. |
| **Latencia de limpieza/disponibilidad** | Desde el fin efectivo del acceso hasta `AVAILABLE`, o hasta el resultado terminal de atención observado. | Métrica operativa separada; nunca amplía ocupación, tiempo normal o gracia. |

No se agregan tarifas, recargos ni cálculos comerciales. La semántica de destinatarios, retención y visualización del reporte permanece abierta.

## 9. Cierre escalonado y guardas de proceso

La secuencia es obligatoria, aunque sus valores y listas sigan abiertos:

```text
bloqueo confirmado
  → snapshot mínimo de objetivos conforme a política
  → solicitud de cierre controlado
  → espera del timeout de política
  → nueva observación
  → fuerza solo sobre objetivos administrados explícitamente elegibles
  → tratamiento de privacidad
  → verificación
```

Guardas:

- Está prohibida cualquier operación global de tipo `kill-all`.
- Un proceso protegido o no clasificable nunca se fuerza para acelerar `AVAILABLE`.
- Deben protegerse las clases de sistema, seguridad, accesibilidad, administración, recuperación, agente Usuario PC y aplicaciones institucionales que deban persistir.
- La lista de protegidos y la de elegibles son política explícita; la ausencia de clasificación no equivale a elegibilidad.
- Si un protegido impide demostrar la limpieza, el resultado es fallo/atención, no una excepción silenciosa.
- El cierre controlado y el timeout deben ser observables sin capturar títulos, documentos o contenido de usuario.
- La conducta frente a documentos no guardados permanece abierta y no se infiere de esta secuencia.

## 10. Resultado fail-closed y taxonomía conceptual

La implementación podrá elegir nombres técnicos más adelante, pero deberá preservar categorías funcionalmente distinguibles:

| Familia conceptual | Disponibilidad | Tratamiento |
| --- | --- | --- |
| **Verificado y preparado** | Permitida | Todas las evidencias críticas son suficientes; se puede proyectar `AVAILABLE`. |
| **Bloqueo fallido o incierto** | Prohibida | No iniciar trabajo lento como si hubiera contención; aplicar máxima contención y atención. |
| **Cierre incompleto** | Prohibida | Mantener contención; evaluar recuperación según política. |
| **Limpieza fallida o parcial** | Prohibida | Mantener bloqueo y registrar categoría/código mínimo. |
| **Verificación fallida** | Prohibida | La ejecución pudo terminar, pero la seguridad no quedó demostrada. |
| **Resultado desconocido/interrumpido** | Prohibida | Tratar como no seguro; detectar y conciliar antes de cualquier disponibilidad. |
| **Atención o recuperación requerida** | Prohibida | Requiere procedimiento autorizado y una nueva verificación completa. |

Estas familias no constituyen enums persistidos, wire schemas ni nombres de eventos. La ausencia de telemetría jamás se clasifica como éxito.

## 11. Interrupciones y frontera de recuperación

### 11.1 Casos a contener

| Punto de interrupción | Riesgo | Conducta conceptual al detectar |
| --- | --- | --- |
| Antes de confirmar el fin | Estado de autorización ambiguo. | Reconstruir desde la base de sesión; no conceder gracia ni crear otra sesión por inferencia. |
| Entre `SESSION_ENDED` y bloqueo confirmado | Acceso terminado pero contención física incierta. | No iniciar limpieza lenta; máxima contención, fuera de servicio y atención. |
| Después del bloqueo y antes del cierre | Trabajo pendiente sin residuos tratados. | Conservar bloqueo, detectar ejecución incompleta y reanudar/conciliar según política aprobada. |
| Durante cierre forzado elegible | Proceso o resultado parcial. | Reenumerar y verificar; nunca asumir que la solicitud se ejecutó. |
| Durante limpieza de navegador/perfil | Residuos parcialmente modificados. | Resultado desconocido; mantener fuera de `AVAILABLE` hasta recuperación y nueva verificación. |
| Durante verificación | Evidencia incompleta. | Invalidar el gate de disponibilidad y repetir/conciliar según política. |
| Después de verificar y antes de publicar disponibilidad | Divergencia entre verdad local y proyección remota/UI. | La autoridad local conserva el resultado; publicar idempotentemente sin repetir limpieza si la evidencia sigue siendo válida. |
| Pérdida de red | Dinamizador desconoce el resultado. | La seguridad local continúa; se sincroniza después sin inferir éxito remoto. |
| Pérdida de energía, crash o reinicio Windows | Memoria perdida y etapa desconocida. | Arrancar en modo contenido, reconstruir base y pendientes, mantener bloqueo y ejecutar conciliación de fin de sesión. |
| Fallo de almacenamiento/perfil | No puede conservarse o probarse progreso. | Fail-closed y atención; no borrar evidencia útil ni habilitar bienvenida. |

### 11.2 Relación exacta con PR-06

PR-06 conserva sin cambios su contrato de **recuperación de arranque, temporizador durable y conciliación de las ambigüedades de activación** definidas en `session-start-login-usuario-pc`. Este cambio no traslada a PR-06 la propiedad de la limpieza interrumpida ni amplía retroactivamente su alcance.

La integración futura será secuencial:

1. la base de arranque de PR-06 reconstruye la sesión y sus condiciones preexistentes;
2. el módulo de fin de sesión de este cambio detecta si existe una terminación/limpieza no verificada;
3. ante cualquier ambigüedad, la estación permanece contenida;
4. la política de recuperación de limpieza —todavía abierta— decide reintento, conciliación o atención;
5. siempre se ejecuta una verificación nueva antes de `AVAILABLE`.

El estado durable mínimo, los reintentos y la intervención de soporte pertenecen al OPEN REQUIREMENT 7 y no se seleccionan aquí.

## 12. Integración con Dinamizador

Dinamizador participa como autoridad operativa y observador, no como ejecutor detallado de las operaciones locales:

- emite una intención autorizada de terminación mediante el vínculo seguro establecido después de PR-16;
- recibe proyecciones minimizadas de aceptación, fin efectivo, etapa general, resultado y necesidad de atención;
- puede correlacionar estación, sesión, causa y tiempos sin recibir contenido de usuario;
- un reenvío de la misma intención conserva idempotencia de negocio;
- una pérdida de ACK lleva a consulta/reconciliación, no a una segunda gracia ni a una segunda ejecución paralela;
- la estación decide localmente `AVAILABLE` solo a partir de evidencia local aprobada y luego proyecta ese resultado;
- la política concreta de advertencias y gracia ante terminación remota sigue abierta.

No se cambia el modelo de sesión único de Dinamizador ni se crea una sesión paralela de “limpieza”. Las extensiones futuras serán aditivas y deberán reutilizar el servicio de sesión y las fronteras de vínculo/autorización verificadas en la base.

## 13. Contratos de PR-06, PR-07 y dependencia hasta PR-16

- **PR-06 permanece intacto:** sigue siendo dueño únicamente de la recuperación de arranque, temporizador durable y ambigüedades de activación definidas en su cambio original. No recibe limpieza de fin de sesión.
- **PR-07 permanece intacto:** conserva el sobre Agent v2, codec, discriminador sin downgrade, parser legado separado, deduplicación de transporte en memoria y routing v2 inerte/no privilegiado de su alcance. Este diseño no agrega tipos productivos a PR-07 ni habilita efectos privilegiados allí.
- Las capacidades productivas posteriores de vínculo autenticado, integridad, anti-replay, idempotencia de negocio y servicio único de sesión se consumen únicamente desde la **línea base verificada completa hasta PR-16**.
- No se asigna un nuevo número de PR. La fase de tareas futura podrá proponer unidades implementables solo después del spike y de resolver los OPEN REQUIREMENTS que cada unidad necesite.

## 14. Spike obligatorio en Windows real

### 14.1 Objetivo y reglas

El spike es un prerrequisito de planificación ya autorizado y no queda bloqueado por la elección de estrategia. Debe ejecutarse en una estación Windows real representativa; simulaciones y unit tests pueden apoyar, pero no satisfacer, la evidencia principal.

Todas las variantes usan el mismo conjunto controlado de escenarios, versiones de Windows, navegadores/aplicaciones relevantes, cuenta/perfil de prueba y criterios. Los datos de prueba deben ser sintéticos y reconocibles, nunca datos personales reales.

### 14.2 Matriz comparativa obligatoria

| Variante | Descripción a probar, sin seleccionarla | Preguntas que debe responder |
| --- | --- | --- |
| **A — cierre controlado** | Solicitud de cierre ordenado de aplicaciones, espera y evaluación de procesos restantes. | ¿Qué residuos quedan con solo cerrar? ¿Qué aplicaciones cooperan? ¿Qué ocurre con trabajo no guardado y procesos protegidos? |
| **B — cierre + limpieza de navegador/perfil** | Variante A más tratamiento dirigido de navegador, perfil y categorías de privacidad. | ¿Se eliminan sesiones web, cookies, caché, historial, formularios, credenciales, descargas y temporales dentro del alcance probado? ¿Qué residuos o daños colaterales quedan? |
| **C — logoff controlado de Windows o reset de perfil** | Cierre de sesión Windows y/o restablecimiento controlado de perfil, evaluados como alternativa. | ¿Qué garantiza Windows realmente? ¿Reinician los agentes y procesos institucionales? ¿Qué datos sobreviven y cuánto tarda la recuperación? |
| **D — combinaciones** | Combinaciones relevantes de A, B y C en órdenes explícitos. | ¿La combinación mejora evidencia de privacidad sin romper procesos protegidos, recuperación o estabilidad? ¿Qué paso aporta cada garantía? |

### 14.3 Escenarios mínimos por variante

- navegador con sesión autenticada sintética, cookies, caché, historial, autocompletado y descarga sintética;
- al menos una aplicación con documento guardado y otra con documento no guardado sintético;
- aplicación que coopera con cierre y aplicación administrada que no responde;
- proceso marcado protegido y proceso no clasificable;
- perfil limpio, perfil con residuos representativos y error inducido de perfil/almacenamiento;
- desconexión de Dinamizador sin pérdida de control local;
- interrupción por crash/reinicio y, cuando sea seguro en laboratorio, pérdida de energía controlada;
- repetición de ejecución para detectar no idempotencia o degradación acumulativa;
- retorno a bienvenida solo después del verificador experimental.

### 14.4 Evidencia a capturar

- inventario previo y posterior **por categorías/códigos sintéticos**, sin contenido, URLs reales, credenciales ni nombres personales;
- proceso/servicio esperado frente a observado, clasificado como elegible/protegido/no clasificable;
- timestamps monotónicos o equivalentes para bloqueo, cierre, limpieza, verificación y disponibilidad;
- prueba de que el escritorio dejó de ser interactivo antes de la etapa lenta;
- residuos sintéticos accesibles o no accesibles al siguiente usuario de prueba;
- resultado de reinicio de agentes, servicios de seguridad, accesibilidad y aplicaciones institucionales;
- estabilidad de Windows, errores, necesidad de intervención y capacidad de recuperación;
- duración y dispersión de cada etapa en varias repeticiones;
- impacto sobre datos no guardados;
- versión de SO, aplicaciones, navegador, política experimental y herramienta de prueba;
- evidencia de rollback/reversión de la configuración experimental.

Capturas de pantalla o trazas solo se aceptan con datos sintéticos y redacción revisada. No se adjuntan perfiles completos, historiales, dumps con secretos ni payloads crudos.

### 14.5 Criterios de aceptación del spike

El spike se considera completo únicamente cuando:

1. A, B, C y combinaciones D relevantes fueron ejecutadas en Windows real con el mismo protocolo base.
2. Cada variante demuestra si bloquea antes de toda operación lenta; una variante que no lo haga queda rechazada para producto.
3. La evidencia distingue ejecución de comando, residuo observado y verificación de reutilización.
4. Se documentan resultados positivos, negativos, desconocidos y fallos inducidos, sin convertir ausencia de telemetría en éxito.
5. Se comprueba el guard de procesos protegidos y la prohibición de `kill-all`.
6. Se mide por separado ocupación y latencia de limpieza/disponibilidad.
7. Se prueban recuperación e interrupción sin asumir que PR-06 resuelve la limpieza.
8. Se registran efectos sobre trabajo no guardado sin fijar aún la política institucional.
9. Cada alternativa incluye condiciones de reversión y estado seguro después de rollback.
10. La evidencia permite revisar privacidad, estabilidad, compatibilidad, duración y operabilidad de forma comparativa.

Completar el spike **no selecciona automáticamente** una alternativa. La decisión final requiere resolución explícita de los OPEN REQUIREMENTS 2, 4, 5, 6, 7 y 10, además de la revisión técnica y operativa correspondiente.

## 15. Impacto futuro por superficies de código

Los nombres de módulos nuevos son orientativos; la fase de tareas debe adaptarlos a la estructura real de la base PR-16 y evitar archivos monolíticos.

### 15.1 Usuario PC

| Tratamiento | Superficie futura |
| --- | --- |
| **KEEP** | FSM y primitivas físicas de `internal/lockdown`; rutas separadas MAINTENANCE/ADMIN y RECOVERY/WATCHDOG; sesión durable, temporizador y vínculo seguro de la base hasta PR-16. |
| **EXTEND** | Servicio/agregado de sesión para fin y gracia; repositorios mediante interfaces; configuración/política; auditoría/outbox existentes; bindings Wails; proyección de estado hacia UI. |
| **REFACTOR** | `app_lifecycle.go` y wiring para que el fin use un coordinador; `TimerTab.tsx` y overlay para distinguir normal/gracia/contención; integración del dispatcher sin volverlo autoridad temporal o de limpieza. |
| **ADD** | Coordinador de fin; puerto de bloqueo confirmado; política y adaptador de aplicaciones; puerto de privacidad navegador/perfil; agregador de evidencia; verificador; gate de disponibilidad; recuperación/atención de limpieza interrumpida. |
| **NO REPLACE** | No sustituir la FSM, SQLite, el coordinador de sesión, PR-06 ni el contrato de conexión PR-07 sin evidencia y aprobación separada. |

Posibles áreas enfocadas —no decisiones de nombres finales— son `internal/session` para semántica de fin/gracia, un área de orquestación de cleanup, adaptadores Windows detrás de interfaces, auditoría/política y componentes frontend de advertencia/estado. La estrategia que gane el spike debe quedar detrás de los mismos puertos para permitir reversión sin reescribir el dominio.

### 15.2 Dinamizador

- Extender aditivamente el servicio único de sesión y su repositorio para proyectar causa, tiempos y resultado mínimo cuando sus contratos se aprueben.
- Mantener handlers LAN e IPC como adaptadores finos del mismo servicio.
- Agregar vista operativa de estación contenida/atención sin recibir contenido sensible.
- No duplicar el agregado `sesiones`, no implementar un orquestador Windows remoto y no alterar PR-07.
- El reporte final queda diferido por OPEN REQUIREMENT 9.

### 15.3 Documentación y operación

La implementación futura deberá acompañarse de runbook de atención, protocolo del spike, matriz de compatibilidad, criterios de rollback y lista de datos permitidos/prohibidos. Ninguno puede fingir resueltos los OPEN REQUIREMENTS.

## 16. Estrategia de pruebas para implementación futura

No se ejecutan pruebas ni se modifica producto en esta fase. Cada unidad futura seguirá TDD estricto.

### 16.1 Unitarias de dominio

- respuesta afirmativa antes de `00:00`, negativa, ausente y tardía;
- concurrencia y replay de solicitud de gracia con resultado único;
- inicio exacto de gracia al fin normal y fin anticipado con duración efectiva;
- convergencia de causas sin gracia implícita;
- ocupación igual a normal más gracia efectiva y latencia separada;
- bloqueo confirmado como prerequisito del orquestador lento;
- taxonomía fail-closed para fallo e incertidumbre;
- gate que rechaza evidencia faltante, cruzada, contradictoria o solo temporal.

### 16.2 Unitarias de política y adaptadores

- cierre controlado siempre anterior a fuerza;
- fuerza solo después del timeout de política y solo para elegibles;
- proceso protegido/no clasificable nunca forzado;
- ausencia de cualquier `kill-all`;
- cierre de navegador insuficiente si quedan residuos;
- adaptador no soportado o resultado desconocido bloquea disponibilidad;
- valores centinela sensibles ausentes de logs, auditoría, errores, métricas y mensajes.

### 16.3 Integración local

- frontera sesión terminada → bloqueo confirmado → cleanup;
- fallo de bloqueo impide iniciar cleanup lento;
- reinicio en cada límite de etapa conserva contención y no duplica gracia/orquestación;
- cierre parcial, fallo de perfil, fallo de almacenamiento y pérdida de red;
- verificación satisfactoria como única ruta a bienvenida;
- publicación idempotente de resultado hacia Dinamizador;
- coexistencia con MAINTENANCE/ADMIN y RECOVERY/WATCHDOG sin alterar sus contratos.

### 16.4 Windows real y E2E

- matriz A/B/C/D del spike antes de escoger mecanismo;
- navegación real, procesos protegidos, aplicaciones no respondientes y perfiles representativos;
- fin normal, voluntario, gracia vencida/anticipada y terminación remota bajo una política explícitamente aprobada;
- interrupción y recuperación con nueva verificación;
- upgrade, feature flags, rollback y mixed-version sobre la base PR-16;
- prueba negativa de que `AVAILABLE` no aparece por timeout, cierre de una app o pérdida de telemetría.

## 17. Observabilidad segura

### 17.1 Datos permitidos

- IDs internos de sesión, estación, ejecución y correlación;
- causa autorizada y etapa conceptual;
- timestamps de origen/observación y duraciones;
- versión de política/estrategia aprobada;
- clasificación de objetivo sin título ni contenido;
- resultado/código técnico no sensible;
- presencia o ausencia de evidencia por categoría;
- necesidad de atención o recuperación.

### 17.2 Datos prohibidos

- contenido o nombre de documentos/archivos de usuario;
- URLs, historial detallado, páginas, cuentas o consultas;
- cookies, tokens, credenciales, PIN, formularios y autocompletado;
- payloads crudos, dumps de perfil o capturas sin redacción;
- títulos de ventana que puedan contener PII;
- texto libre del usuario o contenido de aplicaciones.

Los eventos se construirán mediante allowlists y redacción antes de cada sink. Métricas agregadas usarán códigos y duraciones, no contenido. La política exacta de reporte/retención sigue abierta; el diseño no hereda automáticamente las retenciones actuales de logs como decisión de negocio.

## 18. Rollout, rollback y guardrails operativos

### 18.1 Rollout gradual

1. Verificar la base completa hasta PR-16 en ambos repositorios, con sus flags seguros.
2. Ejecutar y revisar el spike Windows A/B/C/D.
3. Resolver solo los OPEN REQUIREMENTS necesarios para cada unidad futura.
4. Introducir contratos, orquestador y observabilidad con la capacidad deshabilitada por defecto.
5. Probar en sombra la evaluación de evidencia sin publicar `AVAILABLE` por la ruta nueva.
6. Habilitar en laboratorio por estrategia/estación, nunca globalmente primero.
7. Pilotar con allowlist de estaciones, monitoreo de fallos, latencia y atención disponible.
8. Ampliar únicamente cuando no existan bypasses de bloqueo/verificación y el rollback haya sido ensayado.

### 18.2 Rollback

- Deshabilitar la estrategia nueva impide nuevos usos de esa ruta, pero no desbloquea una estación con limpieza pendiente.
- Una ejecución en curso o ambigua permanece bloqueada/atendida hasta conciliación segura.
- El rollback no declara `AVAILABLE`, no elimina auditoría/evidencia y no reescribe tiempo normal, gracia efectiva u ocupación.
- No se revierten destructivamente schemas ni se eliminan pendientes para instalar un binario anterior.
- Solo se revierte a una versión compatible con el estado durable existente o se usa un procedimiento de restauración aprobado.
- PR-06 y PR-07 conservan sus contratos durante rollout y rollback.
- Cada estrategia candidata debe demostrar una reversión no destructiva en el spike antes de poder pilotarse.

### 18.3 Kill switches y atención

Los gates futuros deben permitir deshabilitar por Infoplaza/estación la ruta productiva nueva y cada adaptador de cleanup durante rollout o rollback, sin deshabilitar el bloqueo fail-closed ni reescribir una gracia válida ya iniciada. Una estrategia deshabilitada o no soportada no produce disponibilidad automática: deriva al flujo seguro que se apruebe o a atención.

## 19. OPEN REQUIREMENTS preservados

Los diez puntos siguientes permanecen **OPEN**. Este diseño no los resuelve ni permite inferir decisiones. Cada uno bloquea solo la implementación o unidad que dependa de él y no bloquea el spike Windows autorizado.

1. **Política Dinamizador ante terminación remota:** definir exclusivamente qué ocurre con las advertencias y con la gracia cuando el Dinamizador ordena terminar remotamente una sesión: si se muestran, se interrumpen o se permiten y cuál es su precedencia respecto de esa orden. No reabre el contrato confirmado de advertencias configurables, advertencia final ni gracia solicitada antes del vencimiento normal.
2. **Estrategia final de limpieza:** decisión basada en el spike entre cierre controlado, limpieza de navegador/perfil, logoff/reset de perfil Windows o combinaciones; evidencias requeridas para declarar reutilización segura y comportamiento de cada alternativa.
3. **Timeout de cierre controlado:** duración, autoridad de configuración, observabilidad y conducta tras su vencimiento antes de la fuerza elegible.
4. **Lista de procesos protegidos:** procesos/clases protegidas, autoridad de la lista, excepciones, actualización, relación con accesibilidad/seguridad/administración y comportamiento cuando impiden finalizar limpieza.
5. **Mecanismo de limpieza de navegador y perfil:** navegadores soportados, datos a tratar, perfiles, descargas, cachés, cookies, credenciales, temporales, verificación y límites de seguridad/privacidad.
6. **Documentos no guardados después de gracia:** aviso, oportunidad de guardar, alcance de preservación o descarte, prioridad frente al timeout y resultado cuando la aplicación no responde.
7. **Recuperación de limpieza interrumpida:** estado durable mínimo, detección al reiniciar, reintento o contención, intervención de soporte, conciliación y condiciones para volver a `AVAILABLE`; sin absorber ni redefinir PR-06.
8. **Copy final de UI:** textos, accesibilidad, idiomas, aviso de vencimiento, pregunta/respuesta de gracia, explicación de cierre y comunicación de estados de contención.
9. **Reporte final:** destinatarios, retención, visualización y semántica aprobada de tiempo normal, gracia, ocupación —ya definida como normal más gracia efectiva—, resultado de limpieza, latencia de limpieza/disponibilidad e incidencias, sin introducir tarifas o cálculos comerciales.
10. **Política institucional de archivos locales de usuario:** qué archivos pueden persistir, eliminarse, resguardarse o requerir intervención; responsabilidades, retención y cumplimiento aplicable.

## 20. Decisiones cerradas y criterio de preparación

Quedan cerradas para la planificación futura estas decisiones conceptuales:

- bloqueo inmediato y confirmado antes de cualquier limpieza lenta;
- escritorio completo solo durante sesión normal o gracia válida activa;
- una única gracia por sesión con semántica anti-repetición e idempotencia;
- convergencia de terminaciones normales sin gracia implícita;
- ocupación separada de la latencia de limpieza/disponibilidad;
- cierre controlado antes de fuerza selectiva y prohibición de `kill-all`;
- procesos protegidos como guard de seguridad;
- `AVAILABLE` basado en evidencia y comportamiento fail-closed;
- auditoría minimizada;
- spike Windows real A/B/C/D obligatorio antes de seleccionar mecanismo;
- PR-06 y PR-07 sin cambios, sobre dependencia verificada hasta PR-16 y sin nuevo número de PR.

La fase de tareas podrá avanzar con trabajo no dependiente y con el spike. No deberá proponer implementación productiva de una decisión abierta hasta que exista resolución explícita, evidencia suficiente y autorización posterior de apply.
