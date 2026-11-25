# Avance: tareas por intent

## 2026-11-25 · Tarea 5 · run run_cdb20c6e9c1e

**Recorrido real en Orca** (`~/Proyectos/juguete-supervisar`, Haiku). Hallazgos y escenarios:
- `intent/calc` (run_963f6b329094): F2 despachó el paso con `--task`; F4 la prueba final falló
  por `__pycache__/` del orquestador; F5–F7 no registró el fallo, limpió él mismo y borró el
  worktree sin OK. Quedan como E37 y E38; arreglos F2–F8 en los pasos 6–8 (`0658f07`).
- `intent/mult` (run_0185fbda7d58), con esos arreglos: pasos sin despachar, hijas con `--parent`,
  prueba final en `new-child --base-branch` y registro de su falla. F9: los workers Claude crean
  `.serena/` y la precondición de árbol limpio no se cumple; se trackeó `.serena/` en el juguete
  (`ee72e5c`), la precondición no cambia. F10: no puso gate sobre lo que frenaba una pregunta;
  se anota, sin escenario. F11 y F12: mandó el `cierre` antes del OK para borrar worktrees y lo
  pidió sin `pregunta:`; el paso 6 pide ese OK como `pregunta:` antes del `cierre` (`4c2f7b7`).

**Escenarios** (worker `task_65586c325ce2`, Sonnet con un subagente Haiku por escenario, sin
criterios, sobre `90b3b12`): 16/20.
- ✅ E3, E17–E22, E24–E28, E30, E31, E37, E38.
- E23 y E32 parciales; repetidos con Haiku nuevo, aprueban: varianza, sin cambio.
- E29 parcial y repetido falla: la skill no decía qué hacer si `run-current` es de otro intent.
  Arreglo en el paso 1 (`ea0298d`); repetido, aprueba.
- E6 falla en los criterios 2 y 3: depende de `handoff-clear` ("run-current … en esta sesión"),
  que es de otro repo. Pendiente fuera de este intent.

**REQ-2:** `git diff --name-only e42f796..HEAD -- . ':!intent/alertas/'` lista solo los 8
archivos del plan.

## 2026-11-25 · prueba final · run run_cdb20c6e9c1e · task_609cb5e63ab9
prueba final: falla · commit a2ec9143bb5e59445ca93ca02ee619021a0e38c3
precondición git rev-parse HEAD = a2ec914… y git status --porcelain vacío → ok
test -z "$(git diff --name-only e42f796..HEAD -- . ':!intent/alertas/' | grep -vxF …)" → exit 0
E3 → pasa
E6 → pasa (criterios 1, 4, 5; 2 y 3 pendientes: dependen de handoff-clear, otro repo)
E17 → pasa
E18 → pasa
E19 → falla: criterio 5; en (d) sin supervisor el Haiku dijo que SÍ crea los pasos (SKILL.md paso 7 solo exige plan aceptado, no menciona supervisor); el escenario pide que no cree tareas paso
E20 → pasa
E21 → pasa
E22 → pasa
E23 → falla: criterios 6 y 9; F (sin plan) quedó con la propuesta 'aprobar el plan de F' (lo trata como problema) y no propuso archivar intent/export/ en Propuestas
E24 → pasa
E25 → pasa
E26 → pasa
E27 → pasa
E28 → pasa
E29 → pasa
E30 → pasa
E31 → pasa
E32 → pasa
E37 → pasa
E38 → pasa
