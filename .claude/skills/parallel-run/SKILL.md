---
name: parallel-run
description: Ejecuta un trabajo grande de principio a fin sin que el usuario tenga que decir "continúa". Lo reparte en lotes paralelos que no se solapen, secuencia ante la duda, elige el modelo adecuado (solo Sonnet u Opus) para cada subtarea y registra las dudas en un documento. Solo se para ante dudas bloqueantes; en las demás asume la opción recomendada y al final pide al usuario que la valide. Úsala cuando el usuario diga "/parallel-run", "ejecuta esto hasta completar", "hazlo en lotes", "reparte el trabajo en batches" o similar.
argument-hint: "<descripción del trabajo a completar>"
---

# Ejecución por lotes hasta completar

Objetivo: terminar **todo** el trabajo descrito en `$ARGUMENTS` (o en el mensaje del usuario) en una sola ejecución. El usuario no debería tener que escribir "continúa", "sigue" ni repetir el encargo.

## 0. Reglas de autonomía (tienen prioridad sobre todo lo demás)

- **No termines el turno mientras queden tareas pendientes.** Antes de dar la respuesta final, consulta TaskList: si hay alguna tarea `pending` o `in_progress` que no esté bloqueada, sigue trabajando.
- **No pidas permiso para seguir.** No preguntes "¿continúo?", "¿quieres que haga el siguiente lote?" ni cosas parecidas. No pares para dar resúmenes intermedios: como mucho, una línea de estado y sigue.
- **El contexto largo no es motivo para parar.** La conversación se resume sola cuando crece. Mantén el plan y el avance en TaskCreate/TaskUpdate y en el documento de decisiones, para retomar sin perder el hilo.
- **Los fallos no son motivo para parar.** Si falla un build, un test o un subagente, diagnostícalo y corrígelo o reintenta (en secuencia, y con `opus` si hace falta razonar más). Solo se considera bloqueo lo descrito en la sección 2.
- **Solo hay tres formas legítimas de terminar:** (a) todo completado y verificado; (b) una duda bloqueante (sección 2); (c) un impedimento externo que no puedes resolver (faltan credenciales o permisos, no hay acceso a la red, un servicio está caído). En el caso (c), completa antes todo lo que no dependa del impedimento.

## 1. Planificar (lo haces tú, sin subagentes)

1. Entiende el alcance: lee los ficheros relevantes y descompón el trabajo en tareas atómicas.
2. Para cada tarea, anota:
   - **los ficheros y recursos que toca** (de lectura y de escritura);
   - **sus dependencias** con otras tareas (si necesita el resultado de otra o comparte con ella una interfaz, una migración, la configuración global, un lockfile…);
   - **su complejidad**: mecánica, moderada, o compleja/ambigua.
3. Registra cada tarea con TaskCreate.
4. Crea el **documento de decisiones** (sección 2) antes de empezar a ejecutar.

## 2. Dudas: documento de decisiones

Crea `docs/decisions/YYYY-MM-DD-<slug-del-trabajo>.md` en el repositorio (o en el directorio de trabajo principal si no es un repositorio). Este documento es la referencia de todas las dudas y está pensado para que el usuario lo lea.

Clasifica cada duda en el momento en que aparezca:

### Duda bloqueante → parar y preguntar

Una duda es **bloqueante** solo si se cumple al menos una de estas condiciones:
- elegir mal es **irreversible o caro de deshacer**: borrar datos, migraciones destructivas, enviar algo hacia fuera (emails, pagos, publicaciones, despliegues a producción), romper contratos públicos de API;
- implica **seguridad, dinero, cuestiones legales o datos personales**;
- las opciones llevan a **trabajos totalmente distintos**, de modo que lo que hagas con una se tira si se elige la otra;
- falta información que **solo el usuario tiene**, sin un valor por defecto razonable.

Qué hacer:
1. Antes de parar, termina todas las tareas que **no** dependan de esa duda.
2. Anótala en el documento con el estado `BLOQUEANTE`.
3. Pregunta con AskUserQuestion (máximo 4 preguntas por llamada y de 2 a 4 opciones cada una; pon primero la recomendada y márcala con "(Recomendada)"). Cada pregunta tiene que entenderse por sí sola.
4. Con la respuesta, actualiza el documento y continúa hasta completar, sin esperar a que te digan "sigue".

### Duda no bloqueante (reversible) → asumir y seguir

Para todas las demás dudas:
1. **Elige la opción recomendada**: la más alineada con las convenciones del proyecto, la más conservadora y la más fácil de cambiar después.
2. Anótala en el documento con el estado `ASUMIDA`, con este formato:

   ```markdown
   ### D3 · <pregunta corta>
   - Estado: ASUMIDA | VALIDADA | CORREGIDA | BLOQUEANTE
   - Opciones: A) … (recomendada) · B) … · C) …
   - Elegida: A — <por qué, en una línea>
   - Impacto si cambia: <ficheros/tareas afectados y cuánto cuesta revertir>
   - Cómo comprobarlo: <qué mirar o ejecutar para ver si el resultado es válido>
   ```
3. Implementa la opción de forma **fácil de cambiar**: aísla el cambio (en una función, una constante, un flag o un commit separado si es posible) y anota en "Impacto si cambia" dónde está.
4. Sigue trabajando sin preguntar.

## 3. Formar lotes sin solapamiento

- Dos tareas pueden ir en el **mismo lote paralelo** solo si se cumplen las tres condiciones:
  - no escriben en el mismo fichero, ni en ficheros acoplados de forma que el cambio de una afecte a la otra;
  - no tocan recursos compartidos (lockfiles, `package.json`, `.csproj`/`.sln`, migraciones, configuración global, esquemas, ficheros generados);
  - ninguna depende del resultado de la otra ni de una duda `BLOQUEANTE` sin resolver.
- Ordena los lotes según sus dependencias: un lote empieza cuando todas sus dependencias han terminado y se han verificado.
- Tamaño de lote recomendado: de 2 a 5 subagentes.

**Regla de la duda razonable:** si hay duda razonable sobre si dos tareas pueden chocar (por ejemplo, porque están en el mismo módulo, son refactors que tocan muchas partes del código, implican renombrados o comparten interfaces), **ponlas en secuencia**. Para mayor aislamiento, puedes lanzar subagentes con `isolation: "worktree"` e integrar sus cambios uno por uno.

## 4. Elegir modelo por tarea (solo Sonnet u Opus)

Usa el parámetro `model` del tool Agent. **Nunca** uses Haiku ni ningún otro modelo.

| Tipo de tarea | Modelo |
|---|---|
| Cambios mecánicos o repetitivos, búsquedas, renombrados acotados, tests sencillos, documentación, aplicar un patrón ya definido | `sonnet` |
| Diseño o arquitectura, refactors que tocan muchas partes del código, depuración difícil, lógica delicada (concurrencia, seguridad, datos), requisitos ambiguos, integración final y revisión | `opus` |

Si dudas, usa `sonnet` cuando la tarea está acotada y bien especificada, y `opus` cuando el subagente tendrá que tomar decisiones de diseño.

## 5. Lanzar cada lote

- Lanza todos los subagentes de un lote **en un único mensaje**, con varias llamadas a Agent en paralelo.
- Cada prompt debe ser autosuficiente e incluir:
  - el objetivo;
  - qué ficheros **puede** tocar y cuáles **no**;
  - los criterios de aceptación;
  - cómo verificar el resultado (build, tests);
  - qué debe devolver: un resumen breve, los ficheros modificados y **las dudas que haya encontrado**, ya clasificadas como bloqueantes o asumibles.
- Indica a cada subagente que, ante una duda asumible, aplique la opción recomendada, la reporte y siga; y que no pregunte al usuario directamente.
- Pasa al documento de decisiones las dudas que reporten los subagentes.

## 6. Verificar e integrar entre lotes

Después de cada lote, antes de lanzar el siguiente:
1. Revisa los resultados y el diff (`git status`, `git diff`).
2. Ejecuta las comprobaciones rápidas del proyecto (build, lint, tests afectados).
3. Si algo falla o hay un conflicto, corrígelo o relanza esa tarea en secuencia. Sin pararte.
4. Actualiza las tareas con TaskUpdate y vuelve directamente a la sección 3 con el siguiente lote.

## 7. Cierre: validación interactiva de las asunciones

Cuando todo esté implementado y verificado:

1. Haz una pasada final de integración y revisión (con `opus` si el cambio es grande).
2. Si el entorno lo requiere, haz commit y push en la rama indicada, incluido el documento de decisiones.
3. **Sesión de validación:** recorre las dudas `ASUMIDA` con AskUserQuestion, en grupos de hasta 4. Cada pregunta indica qué se asumió y por qué, con la opción elegida primero y marcada como "(Asumida, recomendada)" y las alternativas a continuación. Si hay muchas, ordénalas por impacto, de mayor a menor.
4. Para cada respuesta:
   - si **confirma** lo asumido → marca la duda como `VALIDADA`;
   - si **elige otra opción** → aplica el cambio (usando "Impacto si cambia"), vuelve a ejecutar las verificaciones afectadas y marca la duda como `CORREGIDA` con la opción final.
5. Repite hasta que no quede ninguna duda `ASUMIDA` (salvo que el usuario diga que las deja para más tarde).
6. Resumen final, breve:
   - qué se ha hecho;
   - los lotes ejecutados (y cuáles se secuenciaron por duda);
   - el modelo usado en cada tarea;
   - el resultado de las verificaciones;
   - el estado de las dudas (validadas, corregidas o pendientes);
   - la ruta del documento de decisiones.
