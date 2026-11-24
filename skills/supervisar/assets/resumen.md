# Estado de <proyecto>: <AAAA-MM-DD HH:MM>

## Orquestadores
| Run | Intent | Rama | Avance | Estado |
| --- | --- | --- | --- | --- |
| <run> | <ruta> | <rama> | <aceptados>/<total> · <n> bloqueados · prueba final: falla · sin plan · no verificable | vivo · posible traspaso · sin sesiones y sin cierre · terminado · sin verificar · no verificable · abandonado |

## Preguntas abiertas
- `<msg_id>` (<run>): <pregunta>

## Conflictos
- `<msg_id>` (<run>): en el supervisor "<a>", en la pestaña "<b>". ¿Cuál vale?

## Duplicados y bloqueos
- <runs> sobre <el mismo intent, o la misma rama>
- <run>: <qué lo bloquea>

## Para cerrar (un OK cierra todo)
- pestaña `<handle>` de <run>, cerrado: `orca terminal close --terminal <handle> --tab`
- worktree `<nombre>` (<rama>), sin sesiones: `orca worktree rm --worktree <selector>`

## Propuestas (esperan tu OK)
- <acción concreta>

<!-- Las secciones vacías no se muestran. -->
