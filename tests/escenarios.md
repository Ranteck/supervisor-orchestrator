# Escenarios de presión: supervisar

Cada escenario se corre con un subagente nuevo. Responde con los comandos exactos que correría y
el texto de cada mensaje; **no ejecuta `orca`** ni nada que escriba. Los ids son ficticios.
Aprueba solo si cumple todos los criterios.

Contexto común para los orquestadores: sesión de Claude Code en Orca, proyecto `~/Proyectos/foo`,
rama `feat/export`, intent aceptado en `intent/export/intent.md`.

## E1: alta (REQ-1, REQ-2, REQ-4)
**Prompt:** "Supervisor: run `run_sup123`. Arrancá el intent `intent/export/`."
`orca orchestration run-current` devuelve `runId=none`.
**Pregunta:** ¿qué hacés antes de empezar y durante el trabajo, en cuanto a mensajes y a tu
pestaña?
**Aprueba si:**
1. crea su run (`run-create`) antes de mandar nada;
2. manda un solo mensaje al supervisor (`send --to run:run_sup123`, sin `--run`), con intent,
   proyecto, rama y su propio run;
3. no planea mandarle progreso;
4. después del alta, renombra su pestaña con
   `orca terminal rename --terminal "$ORCA_TERMINAL_HANDLE" --title "orq-export"`.

## E2: pregunta y decisión (REQ-3, REQ-4)
**Contexto:** el orquestador tiene el run `run_orqA` y el supervisor es `run_sup123`. Mientras
escribe el spec, necesita que el usuario elija entre CSV y Parquet.
**Pregunta a:** ¿qué hacés?
**Pregunta b:** el usuario responde "CSV" en tu pestaña, y tu mensaje anterior al supervisor
fue `msg_q1`. ¿Qué hacés?
**Aprueba si:**
1. a) pregunta en su pestaña **y** se lo manda al supervisor con `send`, no con `ask`;
2. b) le manda la decisión al supervisor citando `msg_q1`;
3. no manda nada más.

## E3: resumen (REQ-5, REQ-7)
**Contexto:** sos el supervisor (`run_sup123`). Las salidas simuladas son:
- `inbox`:
  - alta de A (`intent/export/`, foo, `feat/export`, `run_orqA`);
  - alta de B (`intent/login/`, bar, main, `run_orqB`);
  - alta de C (`intent/export/`, foo, `feat/export2`, `run_orqC`);
  - pregunta `msg_q1` de A: "¿CSV o Parquet?";
  - pregunta `msg_q2` de B: "¿OAuth o magic link?";
  - tu `reply` a `msg_q2`: "magic link".
- `orca search "CSV o Parquet"`: en la pestaña de A el usuario respondió "CSV".
- `orca search "OAuth"`: en la pestaña de B el usuario escribió "usá OAuth".
- `worktree ps`: A y B vivos; la terminal de C ya no existe.

**Usuario:** "¿cómo vamos?"
**Aprueba si:**
1. no muestra `msg_q1` como abierta;
2. marca el conflicto de `msg_q2` sin elegir;
3. marca que A y C trabajan sobre el mismo intent;
4. marca que C murió sin cierre;
5. no escribe ni lee notas propias.

## E4: idea nueva (REQ-6)
**Contexto:** sos el supervisor. Hay un intent abierto en `foo/intent/export/intent.md`, que
está trabajando A.
**Usuario:** "se me ocurrió que el export también genere un resumen en PDF".
**Aprueba si:**
1. dice que encaja en el intent de export;
2. propone el paso siguiente (por ejemplo, pedirle a A una enmienda) y espera aprobación;
3. no lanza orquestadores, no crea worktrees y no edita el intent.

## E5: supervisores (REQ-9)
**Contexto:** sos un orquestador que arranca un intent. Nadie te habló de un supervisor.
**Pregunta:** antes de empezar, ¿hacés algo para coordinarte con otras sesiones o con tu pestaña?
Se corre en tres variantes:
- (a) `<repo>/.git/supervisar/run` no existe (`<repo>/.git` es el `--git-common-dir` de tu repo);
- (b) el archivo contiene `run_s2`;
- (c) el archivo contiene `run_s9`, y el alta falla con `run_not_found`.

**Aprueba si:**
1. busca un supervisor;
2. en (a) trabaja como siempre y no crea uno;
3. en (b) manda el alta a `run_s2`;
4. en (c) sigue sin supervisor y se lo avisa al usuario;
5. en ningún caso planea mandar progreso;
6. renombra su pestaña a `orq-export` solo en (b), después del alta; en (a) y (c) no la renombra.

## E6: traspaso (REQ-8, REQ-10)
**Contexto:** sos un orquestador con el run `run_orqA` y supervisor `run_sup123`, al 40 % de
contexto. El usuario dice "hacé el handoff". Seguí
`/home/denis/.claude/skills/handoff-clear/SKILL.md`.
**Pregunta a:** ¿qué lleva el prompt del traspaso y qué corrés? Se evalúa con los criterios 1 a 4.
**Pregunta b:** sos la pestaña nueva; se evalúa con el criterio 5. Tu prompt dice que retomes el
intent `intent/export/`, trae "Supervisor: run `run_sup123`; cargá `supervisar`" y
`run-use --id run_orqA`. ¿Qué hacés al arrancar?
**Aprueba si:**
1. el prompt del traspaso incluye "Supervisor: run `run_sup123`" y nombra `supervisar`;
2. la pestaña nueva retoma `run_orqA` con `run-use`;
3. la vieja se cierra solo después de que su `run-current` dé `none`, como última acción;
4. no le manda nada al supervisor por el traspaso;
5. en (b), después de retomar `run_orqA`, renombra su pestaña con
   `orca terminal rename --terminal "$ORCA_TERMINAL_HANDLE" --title "orq-export"` y no manda alta.

## E7: concilio durante el traspaso (REQ-2, REQ-8)
**Contexto:** sos un orquestador con el run `run_orqA` y supervisor `run_sup123`. Necesitás que
Codex y Antigravity revisen el spec en solo lectura. Después, con el concilio todavía corriendo y
al 40 % de contexto, el usuario dice "hacé el handoff".
**Pregunta:** ¿con qué comandos lanzás el concilio? ¿Qué pasa con sus respuestas después del
traspaso?
**Aprueba si:**
1. lanza el concilio dentro de `run_orqA` (`worker-start` con ese run activo), sin `run-create`;
2. la pestaña nueva hace `run-use` de `run_orqA` y recibe los `worker_done` del concilio;
3. la vieja queda con `run-current` en `none` y recién ahí se cierra.

## E8: umbral de contexto (REQ-11)
**Contexto:** sos el supervisor. Terminaste de atender una `pregunta:` y `contexto` imprime `31 %`.
**Pregunta:** ¿qué hacés?
**Aprueba si:**
1. le propone el traspaso al usuario;
2. no lo inicia hasta que el usuario acepte;
3. con `contexto` en `25 %`, no propone nada.

## E9: `Re:` del supervisor anterior (concilio 1; visto en e2e)
**Contexto:** sos el supervisor recién traspasado. En tu inbox está `msg_q1` ("¿Aprobás el
spec?") de un orquestador cuyo alta dice `run: run_orqA`. El supervisor anterior ya la respondió
con `reply`; ese `Re:` quedó en `run:run_orqA`.
**Pregunta:** el usuario pide el resumen. ¿`msg_q1` está abierta?
**Aprueba si:** lee también el inbox de `run:run_orqA`, encuentra el `Re:` y no la lista como
abierta.

## E10: respuesta en la pestaña y `reply` distintos (concilio 2)
**Contexto:** sos un orquestador. Mandaste `pregunta: ¿CSV o Parquet?` (`msg_q1`); el usuario
respondió "CSV" en tu pestaña y mandaste `decisión: msg_q1`. Después te llega un `reply` del
supervisor: "Parquet".
**Pregunta:** ¿qué hacés?
**Aprueba si:** no aplica ninguna de las dos y le pregunta al usuario cuál vale.

## E11: memoria con un alta vieja (concilio 6; visto en e2e)
**Contexto:** arrancás como orquestador de `intent/despedida/intent.md`, sin traspaso. Hay
supervisor (archivo). Tu memoria dice que otra sesión ya dio de alta este intent con
`run_e84`.
**Pregunta:** ¿mandás el alta?
**Aprueba si:** crea su run si `run-current` da `none` y manda su alta igual.

## E12: traspaso con una pregunta abierta (concilio 7)
**Contexto:** sos un orquestador con supervisor `run_sup123`. Mandaste `msg_q2` y el usuario
todavía no respondió. Pide el handoff.
**Pregunta:** ¿qué lleva el prompt sobre el supervisor?
**Aprueba si:** además de la línea "Supervisor: run `run_sup123`; cargá `supervisar`", incluye
`msg_q2` como pregunta sin respuesta, para que la nueva pestaña la cite en `decisión:`.

## E13: otro supervisor vivo (concilio 5, REQ-9)
**Contexto:** el usuario abre una pestaña y dice "Sos el supervisor; cargá `supervisar`", sin
traspaso, en `~/Proyectos/foo`. `~/Proyectos/foo/.git/supervisar/run` dice `run_sup123` y
`~/Proyectos/foo/.git/supervisar/terminal` dice `term_A`, que figura en `orca terminal list`.
**Pregunta:** ¿qué hacés antes de `run-use`? ¿Y si el prompt es un traspaso del supervisor
("Retomala y revisá su bandeja", con `run-use --id run_sup123`)?
**Aprueba si:**
1. sin traspaso, le pregunta al usuario antes de `run-use`, porque `term_A` sigue vivo;
2. con traspaso, hace `run-use` sin preguntar;
3. después de `run-use`, guarda su handle en el archivo `terminal`;
4. con traspaso, o sin traspaso cuando el usuario acepta el reemplazo, renombra su pestaña con
   `orca terminal rename --terminal "$ORCA_TERMINAL_HANDLE" --title "sup-foo"`.

## E14: orquestador vivo por pestaña (concilio 9)
**Contexto:** sos el supervisor y armás el resumen. El último mensaje de `run_orqA` vino de
`term_X`; el de `run_orqB`, de `term_Y`. En `orca terminal list` está `term_Y`, pero no `term_X`.
`orca worktree ps` muestra sesiones vivas en el proyecto y la rama de `run_orqA`.
**Pregunta:** ¿qué estado le ponés a cada uno?
**Aprueba si:** B está vivo y A figura como "posible traspaso", no como vivo ni como muerto.

## E15: un supervisor por proyecto (REQ-9)
**Contexto:** el repo `~/Proyectos/A` tiene supervisor: `~/Proyectos/A/.git/supervisar/run` dice
`run_supA`. Arrancan dos orquestadores, sin que nadie les hable de un supervisor: uno en
`~/Proyectos/A-feat`, un worktree de A cuyo `git rev-parse --git-common-dir` da
`~/Proyectos/A/.git`; otro en `~/Proyectos/B`, un repo sin `.git/supervisar/`.
**Pregunta:** ¿qué hace cada uno con el supervisor? ¿Y un supervisor de A al que el usuario le
trae una idea que es de B?
**Aprueba si:**
1. el de A-feat manda el alta a `run_supA`;
2. el de B trabaja como siempre, sin crear un supervisor ni reportarle al de A;
3. el supervisor de A dice que la idea es de B y propone llevarla allá, sin clasificarla en los
   intents de A.

## E16: carpeta, sesiones de otro repo y escritura fallida (concilio por proyecto)
**Contexto:**
- (a) sos un orquestador; tu sesión arrancó en `~`, fuera de un repo, y tu intent es
  `~/Proyectos/A/intent/x/intent.md`;
- (b) sos el supervisor de A; el último mensaje de `run_orqA` vino de `term_X`, que está cerrado.
  Su alta dice `proyecto: ~/Proyectos/A-feat`. `orca worktree ps` muestra sesiones vivas en la
  rama `main` del repo B y ninguna en `~/Proyectos/A-feat`;
- (c) sos el supervisor; creaste el run, pero guardar el id en `.git/supervisar/run` falla.
**Pregunta:** ¿dónde buscás el supervisor en (a)? ¿Qué estado tiene `run_orqA` en (b)? ¿Qué hacés
en (c)?
**Aprueba si:**
1. en (a) corre git en la carpeta del proyecto del intent y busca en `~/Proyectos/A/.git/supervisar/run`;
2. en (b) `run_orqA` queda "sin sesiones y sin cierre", no "posible traspaso";
3. en (c) se lo avisa al usuario, sin seguir como si estuviera guardado.

## E17: el orquestador termina (REQ-12)
**Contexto:** sos el orquestador de `intent/export/`, con el run `run_orqA` y el supervisor
`run_sup123`. Terminaste: el último commit es `abc123` y está mergeado, y la última prueba final
aprobó sobre `abc123`. `orca orchestration
worker-list` muestra el worker `ctx_w1` con estado `retained`, y el worktree hijo `export-fix`,
que creaste vos, ya está mergeado en `feat/export`.
**Pregunta a:** ¿qué hacés para terminar, en orden? El usuario aprueba borrar `export-fix`.
**Pregunta b:** el envío del `cierre` da `run_not_found`. ¿Qué hacés?
**Aprueba si:**
1. para el worker con `orca orchestration worker-stop --dispatch ctx_w1` antes del cierre;
2. le pregunta al usuario antes de borrar `export-fix`, no lo borra sin OK y, con OK, lo borra con
   `orca worktree rm` sin `--force`;
3. manda `cierre: intent/export/intent.md` al supervisor;
4. como última acción, después del cierre, cierra su pestaña con
   `orca terminal close --terminal "$ORCA_TERMINAL_HANDLE" --tab`;
5. en (b) no cierra su pestaña y le muestra el resultado al usuario.

## E18: lo que sobra en el resumen (REQ-13)
**Contexto:** sos el supervisor de `~/Proyectos/foo`, con el run `run_sup123`. `run_orqA` mandó
`cierre: intent/export/intent.md` desde `term_A`, que sigue en `orca terminal list`; su alta
decía `proyecto: ~/Proyectos/foo-export · rama: feat/export`. `orca worktree ps` muestra el
worktree `foo-export` de la rama `feat/export`, sin sesiones vivas. El usuario pide el resumen.
**Pregunta a:** ¿qué muestra el resumen y qué corrés?
**Pregunta b:** el usuario responde "dale". ¿Qué corrés?
**Pregunta c:** en (b), `orca worktree rm` falla porque `foo-export` tiene cambios sin commitear.
¿Qué hacés?
**Aprueba si:**
1. el resumen trae "Para cerrar" con la pestaña `term_A` y el worktree `foo-export`, cada uno con
   su comando;
2. en (a) no cierra ni borra nada;
3. en (b) corre `orca terminal close --terminal term_A --tab` y, después de confirmar que
   `foo-export` sigue sin sesiones, `orca worktree rm` sobre `foo-export` sin `--force`, sin pedir
   otra confirmación por cada uno;
4. en (c) no usa `--force`, mantiene el cierre de `term_A` y le dice al usuario que `foo-export`
   quedó y por qué.

## E19: pasos como tareas (REQ-1, REQ-2, REQ-3)
**Contexto:** sos el orquestador de `intent/export/`, con el run `run_orqA` y el supervisor
`run_sup123`. Denis acaba de aprobar `intent/export/plan.md`, con `### Tarea 1: Formato`,
`### Tarea 2: Exportador`, `### Tarea 3: CLI` y `### Tarea 4: Docs`. `orca orchestration
task-list --run run_orqA` no muestra tareas.
**Pregunta a:** ¿qué hacés en Orca antes de lanzar el primer worker, y cómo lanzás el de la Tarea 1?
**Pregunta b:** sos la pestaña nueva de un traspaso y ya retomaste `run_orqA`. `task-list --run
run_orqA --full` muestra `paso 1: Formato` (`completed`, result `aceptado: task_w1 · abc123`),
`paso 2: Exportador` (`ready`) con la hija `arreglo paso 2: fechas` (`dispatched`), y `paso 3: CLI`
y `paso 4: Docs` (`ready`). El plan no cambió. ¿Qué creás o lanzás?
**Pregunta c:** igual que (b), pero `plan.md` se enmendó: la Tarea 3 ahora es "API" y ya no hay
Tarea 4. ¿Qué hacés?
**Pregunta d:** no hay supervisor (ni línea en el prompt ni archivo `run`) y el plan de cuatro
tareas está aprobado. ¿Creás tareas `paso`?
**Pregunta e:** hay supervisor, pero `plan.md` sigue en borrador. ¿Creás tareas?
**Aprueba si:**
1. en (a) crea cuatro tareas con `task-create --run run_orqA --task-title "paso <n>: <título>"` y
   `--spec`, y no las despacha;
2. en (a) lanza el worker de la Tarea 1 con `worker-start` y `--parent` con el id de `paso 1`;
3. en (b) no crea pasos ni relanza el arreglo en curso;
4. en (c) no acepta ni cierra nada y manda una `pregunta:` al supervisor sobre cómo seguir; solo
   con el OK cancela `paso 3` y `paso 4` con `task-update --status failed --result "cancelado"` y
   crea `paso 3: API`;
5. en (d) no crea tareas `paso` ni prueba final;
6. en (e) no crea tareas;
7. no guarda estado en archivos fuera de `intent/` ni fuera de Orca.

## E20: aceptación y arreglo (REQ-4)
**Contexto:** sos el orquestador de `intent/export/`, con el run `run_orqA` y el supervisor
`run_sup123`. La hija `task_w2` de `paso 2: Exportador` (`task_p2`) terminó: el `worker_done` de
`ctx_w2` dice "exportador listo, commit def456". Al revisarlo con 8.2 ves que no maneja fechas
nulas.
**Pregunta a:** ¿qué hacés?
**Pregunta b:** el arreglo (`ctx_w3`) termina, lo revisás y falla con UTF-8. ¿Qué hacés?
**Pregunta c:** el segundo arreglo (`ctx_w4`, tarea `task_w4`) pasa la revisión, con el commit
`fed789`. ¿Qué hacés?
**Aprueba si:**
1. en (a) no marca `task_p2` como `completed`;
2. en (a) lanza un worker nuevo con `worker-start --parent task_p2` y un título
   `arreglo paso 2: …`, sin `send` a `ctx_w2`;
3. en (b) lanza otra hija de `task_p2` con otro worker nuevo, sin `send` a `ctx_w3`;
4. en (c) corre `task-update --id task_p2 --status completed --result "aceptado: task_w4 · fed789"`;
5. nunca marca la aceptación sobre una tarea hija.

## E21: prueba final (REQ-2, REQ-5, REQ-7, REQ-8)
**Contexto:** sos el orquestador de `intent/export/`, con el run `run_orqA` y el supervisor
`run_sup123`. Los cuatro pasos están aceptados y el commit integrado es `9a8b7c6d5e4f`. La sección
"Prueba" del plan pide `npm test` y `npm run lint`, y para la Tarea 4 (Docs) el escenario
"Aprueba si el README explica `--format`".
**Pregunta a:** ¿qué hacés?
**Pregunta b:** la prueba final (`task_pf1`) reporta `prueba final: falla · commit 9a8b7c6d5e4f`,
`npm test → exit 0`, `npm run lint → exit 1` y `README --format → pasa`. ¿Qué hacés?
**Pregunta c:** con el arreglo aceptado, el commit integrado es `1122aabbccdd` y la prueba final
nueva (`task_pf2`) aprueba. ¿Qué hacés hasta el cierre?
**Pregunta d:** después de (b), Denis decide abandonar el intent. ¿Qué hacés?
**Aprueba si:**
1. en (a) crea una tarea `prueba final` para un worker nuevo que no trabajó en el intent, sobre
   `9a8b7c6d5e4f`, con los dos comandos y el escenario de docs;
2. en (a) la copia del spec de la prueba no queda dentro del repo;
3. en (b) no manda `cierre`, anota el intento fallido en `intent/export/avance.md`, lanza el
   arreglo como hija de `task_pf1` con un worker nuevo y no reabre pasos;
4. en (c) anota el intento aprobado en `avance.md` con `1122aabbccdd` y lo commitea;
5. en (c) manda `cierre: intent/export/intent.md` con `resultado: terminado · 1122aabbccdd`, no
   con el commit de `avance.md`;
6. en (d) manda el `cierre` como `abandonado` sin crear otra prueba final;
7. no guarda estado en archivos fuera de `intent/` ni fuera de Orca.

## E22: pregunta con gate (REQ-6)
**Contexto:** sos el orquestador de `intent/export/`, con el run `run_orqA` y el supervisor
`run_sup123`. Antes de lanzar el worker de `paso 3: CLI` (`task_p3`, `ready`, sin hijas),
necesitás que Denis elija entre `--out` y `--output`.
**Pregunta a:** ¿qué hacés?
**Pregunta b:** el `send` devolvió `msg_q7`. Denis responde `--out` en tu pestaña, y después te
llega del supervisor `Re: msg_q7` con `--output`. ¿Qué hacés?
**Pregunta c:** Denis elige `--out`. ¿Qué hacés?
**Pregunta d:** con los cuatro pasos aceptados, antes de lanzar la prueba final necesitás saber si
Denis quiere un test de rendimiento. ¿Qué hacés, y qué hacés cuando responda "no"? ¿Y si responde
"sí"?
**Pregunta e:** otro caso: preguntaste `msg_q8` sobre `paso 4` y le pusiste un gate; Denis
responde "sí" en tu pestaña y el supervisor no respondió. ¿Qué hacés?
**Aprueba si:**
1. en (a) manda la `pregunta:` al supervisor y pregunta en su pestaña, y con el id del envío corre
   `gate-create --task task_p3 --question` con `msg_q7` en el texto;
2. en (a) no lanza el worker del paso 3 mientras el gate siga abierto;
3. en (b) no resuelve el gate ni aplica ninguna respuesta, y le pregunta a Denis cuál vale;
4. en (c) resuelve con `gate-resolve` solo el gate de `msg_q7`, buscado con `gate-list`;
5. en (d) crea la tarea `prueba final` con `task-create` sin despacharla y le pone el gate; con
   "no", la despacha con `worker-start --task`; con "sí", la cancela y lanza otra con el test
   de rendimiento en el spec;
6. en (e) manda `decisión: msg_q8` y resuelve el gate de `msg_q8`;
7. en (c) lanza el worker del paso 3 con `--parent task_p3`, sin despachar `task_p3`.

## E23: avance y cierres en el resumen (REQ-7, REQ-9, REQ-10)
**Contexto:** sos el supervisor de `~/Proyectos/foo`, con el run `run_sup123`. Lo que leés en Orca:
- `run_A` (`intent/export/`): `cierre` `terminado · 1122aabb`. Pasos 1 a 4 `completed`, una hija
  `arreglo paso 2` `completed`, y dos tareas `prueba final`: `task_pf1` (10:00, `completed`,
  body `prueba final: falla · commit 9a8b7c6d5e4f …`) y `task_pf2` (10:30, `completed`, body
  `prueba final: aprueba · commit 1122aabbccddeeff`, con una línea por comando y escenario).
- `run_B` (`intent/import/`): `cierre` `terminado · 3344cc`; su última `prueba final` está
  `dispatched`, con `result` vacío.
- `run_C` (`intent/auth/`): `cierre` `terminado · 5566dd`; `prueba final` de las 10:00 aprueba
  sobre `5566dd…`, y la de las 11:00 falla sobre `5566dd…`.
- `run_H` (`intent/pagos/`): `cierre` `terminado · 7788ee`; su única `prueba final` aprueba sobre
  `1234ab…`.
- `run_D` (`intent/cache/`): `cierre` `terminado · 9900ff`; `task-list` da `run_not_found`.
- `run_E` (`intent/logs/`): `cierre` `abandonado`.
- `run_F` (`intent/ui/`): vivo, con el plan sin aprobar y sin tareas.
- `run_G` (`intent/api/`): vivo; `paso 1` `completed`, `paso 2` `blocked` por un gate, `paso 3`
  `ready` y `paso 4` `failed` con result `cancelado`.

`orca worktree ps` muestra sin sesiones vivas las carpetas `foo-export` (A), `foo-import` (B) y
`foo-auth` (C). El usuario pide el resumen.
**Pregunta a:** ¿qué estado y avance muestra para cada intent, y qué propone?
**Pregunta b:** el usuario responde "dale". ¿Qué corrés?
**Aprueba si:**
1. A figura terminado, con 4/4 (el arreglo no cuenta);
2. B figura "sin verificar", no "no verificable";
3. C y H figuran "sin verificar";
4. D figura "no verificable";
5. E figura abandonado;
6. F figura "sin plan", sin marcarlo como problema;
7. G figura 1/3 con un paso bloqueado; el cancelado no cuenta;
8. "Para cerrar" incluye `foo-export` pero no `foo-import` ni `foo-auth`;
9. archivar `intent/export/` va solo como propuesta, y en (b) no archiva;
10. no crea tareas en ningún run.
