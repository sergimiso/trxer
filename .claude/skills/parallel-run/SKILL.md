---
name: parallel-run
description: Ejecuta un trabajo grande hasta completarlo dividiéndolo en lotes paralelos que no se solapen, secuenciando ante la duda y eligiendo el modelo más eficiente (solo Sonnet u Opus) para cada subtarea. Úsala cuando el usuario diga "/parallel-run", "ejecuta esto hasta completar en paralelo", "hazlo en lotes", "reparte el trabajo en batches" o similar.
argument-hint: "<descripción del trabajo a completar>"
---

# Ejecución paralela por lotes hasta completar

Objetivo: terminar **todo** el trabajo descrito en `$ARGUMENTS` (o en el mensaje del usuario), sin detenerse a pedir confirmación salvo bloqueo real, repartiéndolo en lotes paralelos seguros.

## 1. Planificar (lo haces tú, sin subagentes)

1. Entiende el alcance: lee los ficheros relevantes y descompón el trabajo en tareas atómicas.
2. Para cada tarea anota:
   - **Ficheros/recursos que toca** (lectura y escritura).
   - **Dependencias** con otras tareas (necesita el resultado de otra, comparte interfaz, migración, config global, lockfile…).
   - **Complejidad**: mecánica / moderada / compleja o ambigua.
3. Registra el plan con TaskCreate (una tarea por unidad) para llevar el seguimiento.

## 2. Formar lotes sin solapamiento

- Dos tareas pueden ir en el **mismo lote paralelo** solo si:
  - no escriben en el mismo fichero ni en ficheros que se importen entre sí de forma que el cambio de una afecte a la otra;
  - no tocan recursos compartidos (package.json/lockfiles, .csproj/.sln, migraciones, config global, esquemas, ficheros generados);
  - ninguna depende del resultado de la otra.
- Ordena los lotes topológicamente: un lote empieza cuando todas sus dependencias han terminado y se han verificado.
- Tamaño de lote recomendado: 2–5 subagentes. Más paralelismo no compensa si aumenta el riesgo de conflicto.

### Regla de duda → secuenciar

Si hay **duda razonable** sobre si dos tareas pueden chocar (mismo módulo, refactor transversal, renombrados, interfaces compartidas, orden de ejecución incierto), **ponlas en lotes secuenciales**, no en paralelo. Un poco más de tiempo es preferible a un conflicto o a un estado inconsistente.

Para aislar mejor el trabajo en paralelo sobre código, puedes lanzar subagentes con `isolation: "worktree"` y luego integrar sus cambios uno a uno.

## 3. Elegir modelo por tarea (solo Sonnet u Opus)

Usa el parámetro `model` del tool Agent. **Nunca** uses Haiku ni ningún otro modelo.

| Tipo de tarea | Modelo |
|---|---|
| Cambios mecánicos o repetitivos, búsquedas, renombrados acotados, tests sencillos, documentación, aplicar un patrón ya definido | `sonnet` |
| Diseño/arquitectura, refactors transversales, depuración difícil, lógica delicada (concurrencia, seguridad, datos), requisitos ambiguos, integración final y revisión | `opus` |

Por defecto, ante duda sobre la complejidad, elige `sonnet` para tareas acotadas y bien especificadas y `opus` si el subagente tendrá que tomar decisiones de diseño.

## 4. Lanzar cada lote

- Lanza todos los subagentes de un mismo lote **en un único mensaje** (varias llamadas Agent en paralelo).
- Cada prompt debe ser autosuficiente: objetivo, ficheros que **puede** tocar y los que **no** debe tocar, criterios de aceptación, cómo verificar (build/tests) y qué devolver (resumen breve + ficheros modificados).
- Indica explícitamente a cada subagente que no modifique nada fuera de su alcance asignado.

## 5. Verificar e integrar entre lotes

Tras cada lote, antes de lanzar el siguiente:
1. Revisa los resultados y el diff (`git status`, `git diff`).
2. Ejecuta las comprobaciones rápidas del proyecto (build, lint, tests afectados).
3. Si algo falla o hay conflicto: corrígelo tú o relanza esa tarea **en secuencial** (con `opus` si el fallo era de razonamiento).
4. Actualiza las tareas con TaskUpdate.

## 6. Completar

- Repite hasta que no queden tareas pendientes.
- Pasada final de integración/revisión (con `opus` si el cambio es grande).
- Si el entorno lo requiere, haz commit y push en la rama indicada.
- Resumen final al usuario: qué se hizo, lotes ejecutados (y cuáles se secuenciaron por duda), modelo usado en cada tarea, resultado de las verificaciones y cualquier cosa que quede bloqueada.
