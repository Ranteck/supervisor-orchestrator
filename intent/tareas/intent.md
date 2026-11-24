# Intención: tareas por intent
Autor: Denis Hugo Perafan · Estado: aceptado · Fecha: 2026-11-13

## Problema
Cuando un orquestador manda el `cierre`, el supervisor lo da por terminado porque el orquestador
lo dijo. Cada worker le avisa al orquestador, y el orquestador revisa si realmente está hecho o qué
le falta; pero hacia arriba solo sube un resumen, y dos niveles arriba un resumen puede decir que
salió bien haya terminado o no el trabajo. Tampoco hay una lista de tareas por intent que diga
cuánto falta.

## Resultado esperado
Cada intent tiene una lista de tareas para completar. La máquina de estados de Orca muestra el
avance y los bloqueos, y el "completo" no lo da una persona ni un resumen, sino la prueba final.
- Rige cuando el orquestador reporta a un supervisor; sin supervisor, trabaja como hoy.
- Los pasos del plan se vuelven tareas en el run que el orquestador informó en su alta. Después de
  un traspaso, la pestaña nueva retoma esas tareas en el mismo run, sin duplicarlas. Antes de que
  apruebe el `plan.md`, que el run no tenga tareas es normal.
- El orquestador es el responsable de lo que hacen sus workers, como un jefe de equipo. Que un
  worker termine, y Orca marque su tarea como completa, no alcanza: el paso cuenta como hecho
  cuando el orquestador revisa el resultado, como piden las house rules (8.2), y lo acepta. Si
  falta algo o tiene un bug, abre una tarea nueva enfocada en solucionarlo, con un worker que
  arranca de cero, en vez de seguir con el que lo construyó.
- Con todos los pasos y arreglos aceptados, el orquestador crea la tarea de prueba final, que corre
  un worker distinto de los que implementaron, sobre el commit integrado. Los comandos de prueba
  del plan tienen que salir con 0, y lo que no se automatiza (docs, skills) va como escenarios
  "Aprueba si…" que el worker reporta como pasa o falla. El resultado registra el commit probado y
  esa evidencia. Si la prueba falla, se arregla y se vuelve a probar.
- Las preguntas que necesitan una decisión mía siguen llegando como mensaje `pregunta:`. Si la
  pregunta frena una tarea, el orquestador además le pone un gate de Orca atado al id de esa
  pregunta, y lo resuelve cuando tiene mi respuesta, por `reply` del supervisor o en su pestaña; si
  respondí distinto en los dos lados, recién cuando elijo cuál vale. Así la tarea figura bloqueada
  mientras espera, y el id de la pregunta viaja en el traspaso.
- El orquestador manda el `cierre` como `terminado` solo si la prueba final pasó; como
  `abandonado`, no pide prueba. El supervisor da el intent por terminado solo si encuentra en el
  run la prueba final aprobada con su evidencia; si no, lo muestra como "sin verificar". Si falta
  el run o su resultado, lo muestra como "no verificable", y no se recrean tareas a ciegas.
- La evidencia de la prueba final queda anotada en el registro de avance del plan, en `intent/`.
- En el resumen, el supervisor muestra el avance real de cada intent (pasos aceptados y
  bloqueados) desde Orca.
- Con la prueba final aprobada, archivar el intent sigue pidiendo mi OK.

## Usuarios y sistemas afectados
Yo; orquestadores y supervisor en Claude Code dentro de Orca; los workers. Las tareas y los gates
de Orca. La skill `supervisar`, en sus secciones del orquestador y del supervisor. El `plan.md` de
`ai-native-sdlc`, del que salen las tareas, solo se lee.

## Restricciones
- Usa la máquina de estados que Orca ya trae (tareas, dependencias y gates), sin SQLite ni un
  almacén propio.
- No contradice el intent raíz: no se cambia la skill `orquestar`; la lógica vive en `supervisar`.
- El estado durable sigue en `intent/`; las tareas de Orca son el estado mientras se trabaja.

## Fuera de alcance
- Que el supervisor actúe solo, como ya dice el intent raíz.
- Cambiar la skill `ai-native-sdlc`.

## Cambios
2026-11-24: después del concilio (Codex y Antigravity) se precisa que un paso está hecho cuando el
orquestador lo acepta, y que la prueba final se crea al final porque Orca completa la tarea con el
`worker_done` y no deja sumar dependencias después. También se precisan cómo se resuelven los
gates, los tipos de cierre y la evidencia, el alcance con supervisor, qué pasa si falta el run y
que `ai-native-sdlc` solo se lee. Aprobado por Denis.
