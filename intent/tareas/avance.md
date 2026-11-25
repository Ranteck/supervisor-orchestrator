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
