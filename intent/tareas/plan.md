# Plan: tareas por intent (desde spec.md 2026-11-24 e intent.md 2026-11-13)
Estado: aceptado

> **Para agentes:** SUB-SKILL REQUERIDA: superpowers:executing-plans (método Native). Los pasos
> usan checkboxes (`- [ ]`).

**Objetivo:** que el orquestador con supervisor lleve los pasos del plan como tareas de Orca, acepte
cada paso y cierre solo con una prueba final aprobada, y que el supervisor muestre el avance y
verifique el cierre en el run.

**Arquitectura:** sin código. Se suma texto a las dos secciones de `skills/supervisar/SKILL.md`,
una plantilla para la prueba final y una columna en el resumen. Cada paso es una tarea padre que
nunca se despacha; los workers y arreglos son hijas (`--parent`). Así, aceptar un paso no pisa el
reporte del worker, los arreglos no inflan el total y los gates nunca caen sobre una tarea en curso.

**Stack:** Markdown, Orca CLI 1.4.272, Claude Code.

**Spec:** `intent/tareas/spec.md` (aceptado 2026-11-24).

## Restricciones globales
- Todo vive en "Si reportás a uno" y "Si sos el supervisor"; sin supervisor no cambia nada (REQ-1).
- Solo tareas, dependencias y gates de Orca; sin almacén propio; no se tocan `orquestar` ni
  `ai-native-sdlc` (REQ-2).
- Sin mensajes nuevos: siguen los cuatro del spec raíz.
- Verificado el 2026-11-24 en runs descartables:
  - el cuerpo del `worker_done` queda en `result` (JSON con `body`) de `task-list --full`;
  - `task-update --result` **pisa** el `result` anterior;
  - una tarea no pasa a `dispatched` a mano, sin un worker;
  - `gate-create` bloquea la tarea y `gate-resolve` la devuelve a `ready`; el id de la pregunta
    entra en `--question` y se ve en `gate-list --full`;
  - una tarea padre se completa a mano con hijas ya cerradas;
  - `task-list --run <otro>` no cambia `run-current`.

## Foco de revisión
1. Traspaso a mitad de un arreglo: la pestaña nueva ve el paso abierto y su hija, y no recrea nada.
2. Dos pruebas finales en el run: vale la última creada, no la última aprobada.
3. El commit del `cierre` abreviado o completo: se compara por prefijo.
4. Un gate de una pregunta ya contestada en la pestaña: se levanta con el `decisión:`, no queda colgado.
5. Un plan con pasos que no son de código (docs): la prueba final los cubre con escenarios.

---

### Tarea 1: Escenarios y línea de base (RED)
**Archivos:** modificar `tests/escenarios.md` y `tests/linea-base.md`.

- [ ] Escribir E19–E23 con el formato de E17 (Contexto, Preguntas, "Aprueba si…" numerados), según
  la tabla de tests. El Foco de revisión entra como criterios: 1 en E19, 4 en E22, 5 en E21, y 2 y
  3 en E23. E19 y E21 suman el criterio "no guarda estado fuera de `intent/` y de Orca" (REQ-2).
- [ ] Correr cada uno con un subagente nuevo, **sin** los cambios, y anotar en `linea-base.md` qué
  criterios fallan. Tiene que fallar al menos un criterio nuevo de este spec por escenario; si
  no, el escenario no prueba nada y se reescribe.
- [ ] Commit: `test: add pressure scenarios for tasks per intent`.

### Tarea 2: El orquestador (GREEN de E19–E22)
**Archivos:** modificar `skills/supervisar/SKILL.md` ("Si reportás a uno"); crear
`skills/supervisar/assets/prueba-final.md`.

**Interfaces que produce** (las lee la Tarea 3):
- Paso: cada `### Tarea <n>` del plan; si no tiene, cada ítem de "Orden de trabajo". Se crea con
  `task-create --run <tu run> --task-title "paso <n>: <título>" --spec "<texto del paso>"` al
  aceptarse `plan.md`, o al retomar, solo para los que falten. Un paso nunca se despacha.
- Si los pasos del run no coinciden con el plan (títulos, pasos nuevos o quitados, por ejemplo tras
  una enmienda), el orquestador no acepta ni cierra nada y manda una `pregunta:`. Con el OK, cancela
  los que sobran o cambiaron con `task-update --status failed --result "cancelado"` y crea los
  nuevos; los cancelados no cuentan en el total.
- Worker de un paso: lo de siempre más `--parent <paso>`. Arreglo: otra hija del mismo padre,
  `--task-title "arreglo paso <n>: …"`, con un worker nuevo, nunca un `send` al que lo construyó.
  Si el arreglo también falla, otra hija más.
- Aceptado: `task-update --id <paso> --status completed --result "aceptado: <hija revisada> · <commit>"`,
  después de la revisión 8.2 y sin gates abiertos en el paso.
- Gate: siempre sobre una tarea sin despachar. Si la pregunta frena un paso, va en el paso; si
  frena algo que todavía no lanzaste (la prueba final o un arreglo), creá esa tarea con
  `task-create`, poné el gate y despachala con `worker-start --task <id>` cuando se resuelva:
  `gate-create --task <id> --question "<msg_id>: <pregunta>"`. Se resuelve con
  `gate-resolve --id <gate> --resolution "<respuesta>"`, con el `<gate>` que `gate-list --run <tu run>
  --full --json` asocia al `<msg_id>`, recién con una respuesta válida (la del
  supervisor, la de tu pestaña o, si difieren, la que elija el usuario). Un gate no frena a un worker
  en curso: mientras siga abierto, no aceptes su entrega.
- Prueba final: con todos los pasos aceptados, una tarea `--task-title "prueba final"` con spec de
  `prueba-final.md` (la copia completada va al scratchpad, nunca al repo), para un worker nuevo que no trabajó en el intent, sobre el SHA integrado. La
  primera línea del `body` es `prueba final: aprueba|falla · commit <sha completo>`, seguida de
  `<comando> → exit <n>` o `<escenario> → pasa|falla` por cada chequeo; `--outcome succeeded` solo
  si aprueba. Si falla, los arreglos son hijas de esa prueba final, revisadas con 8.2, y después
  va una prueba final nueva. Los pasos siguen aceptados: reabrirlos pisaría su `result`.
- `abandonado` no lleva prueba final.
- Registro: cada intento, aprobado o no, suma a `intent/<slug>/avance.md` (`intent/avance.md` en el
  intent raíz) un título `## <fecha> · prueba final · run <run> · <tarea>` con el `body`. Se
  commitea después de la prueba, así que su commit no es el probado: vale el SHA del `body`.
- `cierre` `terminado` solo si la última prueba final aprobó, y lleva su SHA.

- [ ] Escribir los pasos 7–9 de "Si reportás a uno" con esas interfaces, y ajustar el paso 6.
- [ ] `prueba-final.md`: SHA a probar (el worker confirma `git rev-parse HEAD` antes de empezar),
  comandos y escenarios del plan, formato del `body` y del `outcome`.
- [ ] Correr E19–E22 dos veces **con** la skill, y repetir E6 y E17 (traspaso y cierre). Si alguno
  falla, ajustar la skill, no el escenario.
- [ ] Commit: `feat: orchestrator tracks plan steps as Orca tasks`.

### Tarea 3: El supervisor (GREEN de E23)
**Archivos:** modificar `skills/supervisar/SKILL.md` ("Si sos el supervisor", paso 4) y
`skills/supervisar/assets/resumen.md`.

- [ ] En el resumen, por cada alta: `task-list --run <su run> --full --json`. Avance: pasos
  `completed` sobre el total de tareas `paso <n>:` sin cancelar, más los pasos `blocked` y, si la
  última prueba final falló, "prueba final: falla". Sin pasos, "sin plan"; con `run_not_found`,
  "no verificable".
- [ ] Estado de un `cierre` `terminado`, con la tarea de título exacto "prueba final" y
  `created_at` más reciente:
  - terminado si está `completed`, su `result.body` empieza con `prueba final: aprueba`, trae
    una línea por chequeo y su SHA empieza con el commit del cierre;
  - "sin verificar" si no hay prueba final, la última está en curso o falló, o su SHA es otro;
  - "no verificable" si `task-list` da `run_not_found` o el `result` de una prueba `completed` no
    se puede parsear; un `result` vacío de una prueba en curso es "sin verificar".

  `abandonado` figura abandonado. El supervisor nunca crea tareas.
- [ ] "Para cerrar" no propone borrar la carpeta de un cierre "sin verificar" ni "no verificable";
  su pestaña sí, como hoy.
- [ ] Archivar un intent terminado va solo en "Propuestas".
- [ ] `resumen.md`: columna Avance y los estados terminado, sin verificar, no verificable y abandonado.
- [ ] Correr E23 dos veces y repetir E3 y E18. Si alguno falla, ajustar la skill.
- [ ] Commit: `feat: supervisor shows real progress and verifies closures`.

### Tarea 4: README y concilio (6.4)
- [ ] Una línea en `README.md` sobre las tareas por intent.
- [ ] Concilio en solo lectura (nivel medio) sobre el diff `e42f796..HEAD`, que además intente
  romperlo con escenarios nuevos; aplicar lo que pida cambio, repetir los GREEN afectados y
  commitear.

### Tarea 5: Prueba final y cierre
2026-11-24: se suman los escenarios del concilio (E24–E32) y un recorrido real en Orca, porque
hasta acá todo fueron respuestas de Haiku leyendo la skill. Aprobado por Denis.
- [ ] Recorrido real en un repo de juguete descartable (fuera de este repo): un supervisor y un
  orquestador en Claude Code con Haiku, siguiendo la skill, sobre un intent chico con un plan de
  dos pasos. Pasa si, en Orca: los dos pasos existen como tareas sin despachar, los workers son
  hijos, cada paso queda `completed` con `aceptado:` después de revisar, la prueba final la corre
  un worker nuevo sobre un árbol limpio, `avance.md` queda commiteado, el `cierre` lleva el commit
  probado y el resumen del supervisor muestra 2/2 y terminado. Anotar lo que falle como escenario
  nuevo antes de arreglarlo.
- [ ] Un worker nuevo corre, sobre el SHA integrado, E19–E32, E3, E6, E17 y E18, más
  `git diff --name-only e42f796..HEAD`, que solo puede listar `skills/supervisar/SKILL.md`,
  `skills/supervisar/assets/prueba-final.md`, `skills/supervisar/assets/resumen.md`,
  `tests/escenarios.md`, `tests/linea-base.md`, `tests/verde.md`, `README.md` e
  `intent/tareas/plan.md` (REQ-2).
- [ ] Anotar la evidencia en `intent/tareas/avance.md` con el SHA probado, commitear y pedir OK para
  el push.

## Tests a escribir
| Test | REQ |
| --- | --- |
| E19 pasos como tareas: plan aceptado, traspaso a mitad de un arreglo, plan cambiado, sin supervisor, plan sin aprobar | REQ-1, REQ-2, REQ-3 |
| E20 aceptación, arreglo con worker nuevo y segundo rechazo | REQ-4 |
| E21 prueba final que falla con 4/4, arreglo, nueva prueba que aprueba; registro de los dos intentos y `cierre` con su SHA; (b) `abandonado` sin prueba final | REQ-2, REQ-5, REQ-7, REQ-8 |
| E22 pregunta con gate en un paso y en la prueba final; respuestas distintas; `decisión:` | REQ-6 |
| E23 resumen: avance 4/4 con arreglo, un paso `blocked`, un run "sin plan"; terminado, prueba en curso, de otro SHA, sin run, abandonado; carpeta sin verificar fuera de "Para cerrar"; archivar como propuesta | REQ-7, REQ-9, REQ-10 |
| `git diff --name-only` con solo los archivos del plan | REQ-2 |

## Riesgos
- Los escenarios no corren Orca: los comandos salen de lo verificado arriba, no de una prueba de
  punta a punta.
- Un paso padre queda `ready` y se ve en `task-list --ready`; no se despacha porque Orca no
  asigna tareas solo.
- Un gate sobre una tarea ya despachada no está probado: el plan nunca lo hace.
- Tocar el paso 4 del supervisor puede romper E3 y E18, y el orquestador, E6 y E17; por eso se
  repiten.
