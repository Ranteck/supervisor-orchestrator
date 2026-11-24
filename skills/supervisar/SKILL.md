---
name: supervisar
description: Usar SIEMPRE al arrancar o retomar como orquestador de un intent ("sos el orquestador", "orquestá", un `intent/…/intent.md`), aunque nadie mencione un supervisor, para saber si hay uno y reportarle. También cuando el usuario pide un supervisor u observador central ("supervisá", "sos el supervisor", "coordiná los orquestadores") o un prompt trae "Supervisor: run".
---

# Supervisar

Cada proyecto (un repo con todos sus worktrees) tiene a lo sumo un supervisor, que coordina sus
orquestadores por Orca. Su estado vive en
`$(git rev-parse --path-format=absolute --git-common-dir)/supervisar/`, el mismo para todos los
worktrees del repo: el archivo `run` tiene el id de su run permanente, que es su dirección, y
`terminal` tiene su pestaña. Corré git en la carpeta del proyecto (`git -C <carpeta>`); fuera de un
repo no hay supervisor.

## Si sos el supervisor

Corrés en una carpeta de tu proyecto y solo ves ese proyecto.

1. **Al arrancar:** si tu prompt no trae `run-use --id` (no venís de un traspaso) y el archivo
   `terminal` nombra otra pestaña que sigue en `orca terminal list`, hay
   otro supervisor vivo: preguntale al usuario si lo reemplazás y, si dice que no, frená. Después,
   si el archivo `run` existe, `orca orchestration run-use --id <id>`. Si no existe, o `run-use`
   da `run_not_found`, `orca orchestration run-create --objective "supervisor"` y guardá el id en
   ese archivo. Por último, guardá tu `$ORCA_TERMINAL_HANDLE` en el archivo `terminal`. Si no
   podés guardar un archivo, avisale al usuario con el id del run y no sigas como supervisor. Con
   eso guardado, renombrá tu pestaña a `sup-<repo>`, con `<repo>` el nombre de la carpeta que
   contiene el `--git-common-dir`, para que el usuario filtre los supervisores por ese prefijo:
   `orca terminal rename --terminal "$ORCA_TERMINAL_HANDLE" --title "sup-<repo>"`. Si falla,
   avisale y seguí.
2. **Altas, decisiones y cierres:** no respondas; quedan en el inbox.
3. **Preguntas:** mostráselas al usuario y mandá su respuesta con
   `orca orchestration reply --id <msg_id> --body "<respuesta>"`.
4. **Resumen, a pedido:** con `assets/resumen.md`, desde
   `orca orchestration inbox --terminal run:<id> --limit 100` de tu run y del run de cada alta (ahí
   quedan los `Re:` de las respuestas que mandaste), `orca worktree ps` y el `intent/` de tu
   proyecto. Un orquestador con `cierre:` está cerrado; si no, está vivo si la pestaña de su último
   mensaje sigue en `orca terminal list`, y si no está ahí, pero la carpeta de su alta tiene
   sesiones vivas en `orca worktree ps`, es un posible traspaso (la pestaña nueva no avisa hasta su próximo mensaje). Una pregunta con
   `Re:` o `decisión:` no está abierta. Si no tiene, buscala con `orca search "<texto>"`:
   - si el usuario la respondió en la pestaña del orquestador, no está abierta;
   - si hay dos respuestas distintas, mostrá las dos y preguntá cuál vale.

   **Avance y cierres:** por cada alta, `orca orchestration task-list --run <su run> --full --json`.
   El avance son los pasos (`paso <n>:`) `completed` sobre el total sin los cancelados (`failed`
   con `cancelado`), más los `blocked` y, si la última prueba final falló, "prueba final: falla";
   sin pasos es "sin plan", que no es un problema, y con `run_not_found`, "no verificable". Un
   `cierre` `terminado` figura terminado solo si la tarea de título exacto `prueba final` con el
   `created_at` más reciente está `completed`, la primera línea de su `result.body` es
   `prueba final: aprueba · commit <sha>`, sigue una línea por chequeo y ese sha empieza con el
   commit del cierre. Si no hay prueba final, la última está en curso o falló, o es de otro commit,
   figura "sin verificar"; si `task-list` da `run_not_found` o el `result` de una prueba
   `completed` no se puede leer, "no verificable". Un `abandonado` figura abandonado. Nunca crees
   tareas, y archivar un intent terminado va solo en "Propuestas".

   En "Para cerrar" van las pestañas de orquestadores con `cierre:` que siguen en
   `orca terminal list` y la carpeta de su alta, salvo la principal del repo, si no tiene sesiones
   vivas en `orca worktree ps` y su intent no figura "sin verificar" ni "no verificable"; cada una con su comando. Con un solo OK del usuario, corré todos
   sin otra confirmación por cada uno; antes de borrar una carpeta, confirmá que sigue sin
   sesiones, y usá `worktree rm` sin `--force`. Si alguno falla, seguí con el resto y decí cuál
   quedó y por qué.
5. **Ideas nuevas:** decí a qué intent de tu proyecto van o si piden uno nuevo, y proponé el paso
   siguiente; si son de otro proyecto, decilo y proponé llevarlas al supervisor de ese proyecto. Para
   abrir un orquestador aprobado, usá `handoff-clear` con la línea
   "Supervisor: run `<id>`; cargá `supervisar`".
6. Todo se reconstruye de esas fuentes: no guardes notas propias. Nada se lanza, cierra ni
   redirige sin aprobación del usuario; con su OK, una instrucción a un orquestador va con
   `orca orchestration send --to run:<su run> --subject "…" --body "…"`.

## Si reportás a uno

Hay supervisor si el prompt trae "Supervisor: run `<id>`" o si existe el archivo `run` de tu
repo; si difieren, vale el archivo, y tu memoria no cuenta. Si no hay, trabajá como siempre y no crees uno.

1. **Tu run:** si `orca orchestration run-current` da `none`,
   `orca orchestration run-create --objective "<intent>"`. Es tu único run: los workers y concilios
   van en él, porque el traspaso se lleva un solo run.
2. **Mensajes:** solo estos cuatro, con
   `orca orchestration send --to run:<supervisor> --subject "…" --body "…"`, sin `--run`, con
   este asunto y cuerpo exactos:

   | Cuándo | `--subject` | `--body` |
   | --- | --- | --- |
   | al arrancar, o al retomar sin traspaso, aunque tu memoria diga que otra sesión ya lo dio de alta | `alta: <ruta del intent>` | `proyecto: <ruta> · rama: <rama> · run: <tu run>` |
   | cada decisión que le pedís al usuario, una por mensaje, antes de hacérsela en tu pestaña | `pregunta: <la pregunta, en una línea>` | `opciones: <a, b, …> · contexto: <archivo donde se decide>` |
   | apenas el usuario responde en tu pestaña, una por pregunta | `decisión: <id del mensaje de la pregunta>` | `<respuesta del usuario, textual>` |
   | terminaste o abandonaste el intent | `cierre: <ruta del intent>` | `resultado: terminado o abandonado · <commit o motivo>` |

   En tu pestaña, preguntá con texto y terminá el turno, sin AskUserQuestion: bloquea la pestaña y
   no deja entrar la respuesta del supervisor.
3. Si un envío da `run_not_found`, seguí sin supervisor y avisale al usuario. Con el alta enviada
   sin ese error, o al retomar tu run en un traspaso, renombrá tu pestaña a `orq-<slug>`, con
   `<slug>` la ruta del intent entre `intent/` y `/intent.md` con `-` en vez de `/` (`<repo>`, como
   en `sup-<repo>`, si es el intent raíz), para que el usuario sepa qué intent lleva cada pestaña:
   `orca terminal rename --terminal "$ORCA_TERMINAL_HANDLE" --title "orq-<slug>"`. Si falla,
   avisale y seguí.
4. La respuesta del supervisor te llega como mensaje y te despierta: aplicala, sin mandar
   `decisión`. Si en tu pestaña el usuario respondió otra cosa, no apliques ninguna y preguntale
   cuál vale.
5. **Traspaso:** con `handoff-clear`; el prompt lleva "Supervisor: run `<id>`; cargá
   `supervisar`" y los ids de las preguntas sin respuesta. Al supervisor no le avisás.
6. **Al terminar o abandonar el intent**, antes del `cierre`: pará con
   `orca orchestration worker-stop --dispatch <id>` cada worker que
   `orca orchestration worker-list --run <tu run>` muestre sin `released`, y proponé borrar los
   worktrees que creaste y ya están mergeados en tu rama; borralos solo con OK, con
   `orca worktree rm --worktree <selector>` sin `--force`. Si el `cierre` sale sin
   `run_not_found`, como última acción cerrá tu pestaña con
   `orca terminal close --terminal "$ORCA_TERMINAL_HANDLE" --tab`, y si no podés, decíselo al
   usuario; si dio `run_not_found`, no la cierres: mostrale el resultado al usuario.
   El `cierre` `terminado` va solo si la última prueba final aprobó (paso 8), con su commit;
   `abandonado` no la pide.
7. **Pasos del plan:** con `plan.md` aceptado, cada `### Tarea <n>` (si no hay, cada ítem de
   "Orden de trabajo") es un paso. Creá los que falten en
   `orca orchestration task-list --run <tu run> --full --json`, también al retomar, con
   `orca orchestration task-create --run <tu run> --task-title "paso <n>: <título>" --spec "<texto del paso>"`.
   Antes, no crees ninguno. Un paso nunca se despacha: su worker lleva `--parent <id del paso>` en
   el `worker-start`. Si los pasos del run no coinciden con el plan (títulos, pasos nuevos o
   quitados), no aceptes ni cierres nada y mandá una `pregunta:`; con el OK, cancelá los que
   sobran o cambiaron con `task-update --id <paso> --status failed --result "cancelado"` y creá
   los nuevos.
   - **Aceptar:** que el worker termine no alcanza. Revisá su entrega (8.2 de las house rules) y,
     si la aceptás y el paso no tiene un gate abierto,
     `task-update --id <paso> --status completed --result "aceptado: <tarea hija> · <commit>"`.
     Si no, lanzá otra hija con `--parent <paso> --task-title "arreglo paso <n>: <qué>"` para un
     worker nuevo, nunca un `send` al que lo construyó; si el arreglo falla, otra hija más.
8. **Prueba final:** con todos los pasos aceptados, la lanzás vos, en tu run: completá
   `assets/prueba-final.md` en tu scratchpad, nunca en el repo, y corré `worker-start` con
   `--task-title "prueba final"` y esa copia como spec, para un worker nuevo que no trabajó en el
   intent. Con cada `worker_done` de una prueba final, apruebe o falle, sumá a `avance.md` en la
   carpeta del intent (`intent/avance.md` en el raíz) un título
   `## <fecha> · prueba final · run <run> · <tarea>` con su `body`, y commitealo. Si falla, los arreglos son hijas de esa prueba final,
   con workers nuevos y revisados como un paso, y después va una prueba final nueva; los pasos no
   se reabren, porque `task-update` pisa su `result`.
9. **Preguntas que frenan:** si una `pregunta:` frena algo, poné el gate sobre una tarea sin
   despachar. Si frena un paso, va en el paso, y su worker sigue yendo con `--parent`. Si frena
   algo que todavía no lanzaste (prueba final o arreglo), creá esa tarea con `task-create`, ponele
   el gate y, cuando se resuelva, lanzala con `worker-start --task <id>`; si la respuesta cambia
   lo que tiene que hacer, cancelala (`failed`, `cancelado`) y lanzá otra con el spec completo,
   porque el spec de una tarea no se edita. Con el id del envío:
   `orca orchestration gate-create --task <id> --question "<msg_id>: <pregunta>"`. Mientras siga
   abierto, no aceptes ni lances lo que frena. Con una respuesta válida (la del supervisor, la de
   tu pestaña o, si difieren, la que elija el usuario), buscá el gate de ese `<msg_id>` en
   `orca orchestration gate-list --run <tu run> --full --json` y resolvé solo ese con
   `orca orchestration gate-resolve --id <gate> --resolution "<respuesta>"`.

## Contexto

Al terminar cada mensaje (supervisor) o cada tarea (orquestador), corré
`python3 ~/.claude/skills/handoff-clear/scripts/contexto`. Con 30 % (supervisor) o 40 %
(orquestador) o más, proponé el traspaso con `handoff-clear`. El supervisor pasa su run como la
orquestación en curso (el prompt lleva `run-use --id <tu run>`) y dice "Sos el supervisor; cargá
`supervisar`" en vez de la línea "Supervisor: run".
