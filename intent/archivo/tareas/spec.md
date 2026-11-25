# Spec: tareas por intent (desde intent.md 2026-11-13, enmendado 2026-11-24)
Estado: aceptado

Origen: `RE n` es la viñeta n de "Resultado esperado" del intent.

## Requisitos
- REQ-1 MUST: Rige solo con supervisor; sin él, el orquestador trabaja como hoy. Origen: RE 1.
- REQ-2 MUST: Usa solo tareas, dependencias y gates de Orca, sin almacén propio; lo durable queda en `intent/` y solo
  cambia `supervisar`. Origen: Restricciones; Fuera de alcance.
- REQ-3 MUST: Con `plan.md` aceptado, cada paso es una tarea en el run del alta; antes, sin tareas
  es normal. La pestaña de un traspaso sigue con ellas sin duplicarlas. Origen: RE 2.
- REQ-4 MUST: Un paso está aceptado solo cuando el orquestador revisó (8.2) y aceptó su entrega o
  la de su arreglo, y el run lo distingue de un worker que solo terminó. Si no la acepta, abre una
  tarea de arreglo para un worker nuevo. Origen: RE 3.
- REQ-5 MUST: Con todo aceptado, el orquestador crea la tarea de prueba final: sobre el commit
  integrado, un worker que no implementó corre los comandos de prueba del plan (todos salen con 0)
  y sus escenarios "Aprueba si…" (todos pasan), y registra commit y evidencia. Si falla, arreglo
  y otra prueba. Origen: RE 4.
- REQ-6 MUST: Si una `pregunta:` frena una tarea, un gate atado a su id la bloquea hasta una
  respuesta válida, por el supervisor o en la pestaña (si difieren, la que elija el usuario); se
  levanta solo ese gate. El id viaja en el traspaso. Origen: RE 5.
- REQ-7 MUST: `cierre` `terminado` solo con la prueba final aprobada; `abandonado` no la pide. El
  supervisor muestra terminado solo si la última prueba del run aprobó sobre el commit del cierre;
  si no, "sin verificar"; sin run o resultado legible, "no verificable", sin recrear tareas.
  Origen: RE 6.
- REQ-8 MUST: La evidencia queda en un registro de avance en la carpeta del intent, sin editar
  `plan.md`. Origen: RE 7; Usuarios, "solo se lee".
- REQ-9 MUST: El resumen muestra por intent, desde Orca, los pasos del plan aceptados sobre el
  total y los bloqueados. Origen: RE 8.
- REQ-10 MUST: Archivar sigue pidiendo el OK del usuario. Origen: RE 9.

## Escenarios
### Tareas (REQ-1, REQ-3)
- GIVEN un plan sin aprobar, WHEN se pide el resumen, THEN el run vacío no es un problema.
- GIVEN un plan aceptado de 4 pasos, WHEN hay traspaso, THEN siguen las mismas 4 tareas.
- GIVEN ningún supervisor, WHEN el orquestador ejecuta su plan, THEN trabaja como hoy.

### Aceptación (REQ-4, REQ-9)
- GIVEN el paso 2 de 4 con un bug, WHEN se revisa, THEN no figura aceptado y lo arregla otro
  worker; aceptado el arreglo, el resumen dice 4/4.

### Prueba final (REQ-5, REQ-7, REQ-8)
- GIVEN todo aceptado, WHEN un comando sale distinto de 0, THEN no aprueba ni hay `cierre`
  `terminado`; tras el arreglo, otra aprueba y deja commit y evidencia en el run y el registro.

### Pregunta (REQ-6)
- GIVEN una tarea frenada por una pregunta con dos respuestas distintas, WHEN el usuario elige,
  THEN se levanta ese gate y la tarea queda lista si nada más la bloquea.

### Cierre (REQ-7, REQ-10)
- GIVEN un `cierre` `terminado`, WHEN se pide el resumen, THEN figura terminado si la prueba
  aprobó sobre ese commit; "sin verificar" si falló, falta o es de otro; "no verificable" sin run.
  Un `abandonado` figura abandonado, y nada se archiva sin OK.

## Concerns
- Orca completa la tarea con `worker_done` y no tiene estado "aceptado": lo resuelve el plan.
- A verificar en Orca: `worker_done` en `result` de `task-list --full`, `task-update` sobre
  `completed`, gates sobre `dispatched`, `task-list --run` sin cambiar el run activo, y el id de
  la pregunta en `gate-create`.
- REQ-8 lee "registro de avance del plan" como registro aparte (confirmado por Denis, 2026-11-24).
- Riesgos aceptados: prueba reportada sin correr (la frena 8.2), traspaso con gate abierto,
  tareas fallidas previas al plan, pregunta previa al plan sin gate.
