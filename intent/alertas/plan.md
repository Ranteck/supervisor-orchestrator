# Plan: alertas (desde spec.md 2026-11-25 e intent.md 2026-11-25)
Estado: aceptado

> **Para agentes:** SUB-SKILL REQUERIDA: superpowers:executing-plans (método Native). Los pasos
> usan checkboxes (`- [ ]`).

**Objetivo:** que el supervisor muestre en el resumen los avisos `✦` de las pestañas de sus
orquestadores y los vuelva a mirar antes de cerrarlas; que el orquestador pida OK antes de cerrar
su pestaña y, en un traspaso, pida mirar los avisos y pase los charlados.

**Arquitectura:** texto en las dos secciones de `skills/supervisar/SKILL.md`, una sección en
`assets/resumen.md` y un script chico, `skills/supervisar/scripts/avisos`, que es el único
lugar donde se filtra la pantalla (REQ-3), con su test. Sin tocar `handoff-clear` ni
`orquestar`. Se implementa en el worktree `feat/alertas` (desde `main`), porque la skill es un
symlink en cuatro perfiles y pushear a `main` la publica.

**Stack:** Markdown, Python 3 (biblioteca estándar), Orca CLI, Claude Code.

**Spec:** `intent/alertas/spec.md` (aceptado 2026-11-25, REQ-7 enmendado ese día).

## Restricciones globales
- Todo vive en "Si sos el supervisor" y "Si reportás a uno"; sin supervisor no cambia nada (REQ-1).
- Sin mensajes nuevos: siguen los cuatro (REQ-8).
- Verificado el 2026-11-24/25:
  - el aviso es un bloque: `✦ Heads up · <texto>`, a veces con continuación sangrada, el pie
    `1: Learn more   2: Knew this already   0: Dismiss` y, debajo, el separador `───` del prompt;
  - un turno nuevo lo borra, también el que abre un mensaje de Orca (cuatro casos);
  - `orca terminal read` sin `--screen` puede devolver la salida acumulada (`source: stream`, visto
    en una pestaña): ahí quedaría un aviso ya borrado. Con `--screen --json`, `source: screen` y lo
    que el usuario está escribiendo va aparte, en `draft`, fuera de `tail`.

## Interfaces
- **`avisos`:** `orca terminal read --terminal <t> --screen --json | python3 <skill>/scripts/avisos`.
  Lee el JSON por stdin y escribe solo el texto de los avisos, uno por línea, o
  `no se pudo leer` si el JSON no se puede parsear, no trae `result.terminal.tail` como lista de
  textos (también un error de Orca sin `result`) o `source` no es `screen`. Solo mira `tail`,
  nunca `draft`; une las filas partidas por el ancho con `tailRows` cuando venga. Un bloque vale
  solo completo y pegado al prompt activo: empieza al inicio de línea con `✦ Heads up · `, sigue
  con hasta tres líneas sangradas, termina en el pie exacto y debajo vienen un separador (línea
  entera de `─` desde la columna 0), la línea del prompt (`❯`) y otro separador. Así no pasa una
  cita del aviso en el chat. Si falta algo, lo descarta sin error (formato cambiado o pestaña muy
  angosta: riesgo aceptado). La salida sin filtrar nunca entra al contexto. La skill lo nombra
  relativo a su carpeta, como `assets/resumen.md`.
- **Pestañas a leer (REQ-2, REQ-4):** por cada alta, con o sin `cierre:`, el remitente (`from`)
  de su último mensaje en el inbox del supervisor (`run:<supervisor>`), solo si sigue en
  `orca terminal list`. Nunca la propia ni los remitentes de los inbox de los runs de las altas
  (respuestas y workers). Cada pestaña se lee una vez aunque la compartan dos altas.
- **Resumen:** sección "Avisos" en `resumen.md`: `` - `<pestaña>` (<intent>): <aviso> ``,
  `` - `<pestaña>` (<intent>): no se pudo leer `` y, para un posible traspaso (sin pestaña
  conocida), `- <run> (<intent>): avisos sin leer (posible traspaso)`; el estado en
  "Orquestadores" sigue siendo `posible traspaso` (E14). El supervisor no responde ni descarta
  nada.
- **Cierre (REQ-5):** tras un `cierre` sin `run_not_found`, el orquestador pregunta en su pestaña
  "¿Cierro esta pestaña?", sin `pregunta:` ni AskUserQuestion, y termina el turno. Es OK un sí sin
  condiciones ("sí", "dale", "cerrala"); no lo son `1`, `2`, `0`, charlar un aviso, un sí con
  condiciones ni un sí mezclado con otro pedido. Si no es OK, hace lo que le pidieron y vuelve a
  preguntar al terminar ese turno.
- **Para cerrar (REQ-6):** el OK vale solo para los ítems de esa lista. Antes de cada
  `terminal close`, corre `avisos`; si da un aviso que no figuraba en el resumen, o
  `no se pudo leer`, no la cierra, lo muestra, pregunta y sigue con el resto. Con un OK nuevo a
  esa pregunta, vuelve a correr `avisos`: la cierra si da solo avisos ya mostrados o ninguno; con
  otro aviso o `no se pudo leer`, la deja de nuevo y, si la lectura sigue fallando, dice que la
  cierre a mano. Las carpetas siguen como hoy (E18, E23).
- **Traspaso (REQ-7):** al proponer el traspaso (sección "Contexto"), el orquestador con
  supervisor pide mirar los avisos de su pestaña antes de aprobar. El prompt de `handoff-clear`
  suma, si los hay, los avisos que el usuario charló, en texto propio, sin `✦` ni secretos, con
  pestaña e intent de origen, y la pestaña nueva los muestra en su primera respuesta, junto a sus
  tres líneas. Si el usuario pide el traspaso sin propuesta, ya vio la pantalla al escribirlo.

## Foco de revisión
1. Un `✦` en el chat o en `draft`, un bloque sin pie y uno sin separador: `avisos` no los deja pasar.
2. La pregunta de cierre respondida con `1` o con "sí, pero antes…": no es un OK.
3. Una pestaña aprobada en "Para cerrar" que mientras tanto mandó otro mensaje: se relee igual.
4. Traspaso sin avisos charlados: el prompt no inventa ninguno.

---

### Tarea 1: Tests y escenarios (RED) · REQ-1 a REQ-8
**Archivos:** crear `tests/test_avisos.py` y `tests/pantallas/` (JSON inventados); modificar
`tests/escenarios.md` y `tests/linea-base.md`.

- [ ] `test_avisos.py` (unittest) con pantallas inventadas: aviso de una y de dos líneas; un token
  falso arriba del aviso; `✦` en el chat y en `draft`; una cita completa del aviso en el chat,
  encima de un prompt real sin aviso; un borde sangrado de `─`; bloque sin pie, sin separador o
  con más de tres líneas; pantalla angosta con filas partidas (`tailRows`); `source: stream` y
  `screen-unavailable`; error de Orca sin `result`; `tail` que no es lista de textos; JSON roto.
  Verlo fallar sin el script.
- [ ] E33–E36 con el formato de E17, según la tabla. Actualizar E17 (pregunta antes de cerrar) y
  E18 (corre `avisos` antes de cerrar `term_A`), porque el spec cambia ese comportamiento (6.2).
- [ ] Correr E33–E36, E17 y E18 con subagentes nuevos **sin** los cambios y anotar qué criterios
  fallan; si ninguno falla, el escenario se reescribe.
- [ ] Commit: `test: add alert scenarios and filter tests`.

### Tarea 2: `avisos` (GREEN del test) · REQ-3
- [ ] `skills/supervisar/scripts/avisos`, ejecutable, solo biblioteca estándar; correr
  `python3 -m unittest tests/test_avisos.py` en verde.
- [ ] Commit: `feat: add alert filter for tab screens`.

### Tarea 3: El orquestador (GREEN de E34, E36 y E17) · REQ-1, REQ-5, REQ-7, REQ-8
**Archivos:** `skills/supervisar/SKILL.md` ("Si reportás a uno", pasos 5 y 6, y "Contexto").

- [ ] Pregunta de cierre en el paso 6; avisos charlados en el prompt y en la primera respuesta de
  la pestaña nueva en el paso 5; pedir mirar los avisos al proponer, en "Contexto".
- [ ] Correr E34, E36 y E17 dos veces y repetir E6 y E12; anotar en `tests/verde.md`.
- [ ] Commit: `feat: orchestrator asks before closing and carries discussed alerts`.

### Tarea 4: El supervisor (GREEN de E33, E35 y E18) · REQ-2, REQ-3, REQ-4, REQ-6, REQ-8
**Archivos:** `SKILL.md` ("Si sos el supervisor", paso 4) y `assets/resumen.md`.

- [ ] Pestañas a leer, sección "Avisos" y relectura en "Para cerrar", con el comando de `avisos`.
- [ ] Correr E33, E35 y E18 dos veces y repetir E3, E14 y E23; anotar en `tests/verde.md`.
- [ ] Commit: `feat: supervisor shows tab alerts and rechecks before closing`.

### Tarea 5: Spec raíz, README y concilio (6.4)
- [ ] Enmendar en `intent/spec.md`, con "(cambiado 2026-11-25, aprobado por Denis)": REQ-3 (la
  pregunta de cierre no va al supervisor), REQ-8 (el traspaso pide mirar los avisos), REQ-12
  (cierra tras el OK) y REQ-13 (relee antes de cerrar); y REQ-10, que en un traspaso también
  pasan los avisos charlados. Una línea en `README.md`.
- [ ] Commit: `docs: amend root spec and README for alerts`.
- [ ] Concilio en solo lectura (nivel medio) sobre `d30be49..HEAD`, que intente romperlo con
  escenarios nuevos; aplicar lo que pida cambio, repetir los GREEN afectados y commitear.

### Tarea 6: Prueba final y merge
- [ ] Un worker nuevo, sobre un árbol limpio en el último commit de `feat/alertas`, corre:
  `python3 -m unittest tests/test_avisos.py`; E33–E36, E3, E6, E12, E14, E17, E18 y E23;
  `avisos` sobre todas las pestañas vivas (solo avisos o nada); y
  `git diff --name-only d30be49..HEAD`, que solo puede listar `skills/supervisar/`, `tests/`,
  `README.md`, `intent/spec.md` e `intent/alertas/`.
- [ ] Recorrido real en un repo de juguete descartable, como `tests/e2e-orca.md`: supervisor y
  orquestador Haiku; resumen, pregunta de cierre respondida con `1` y después con "dale", "Para
  cerrar" y un traspaso propuesto. Si hay un aviso visible, sale en el resumen. Los perfiles
  cargan la skill de `main` por symlink: cada participante (supervisor, orquestador, la pestaña
  nueva del traspaso) lee `SKILL.md` y `scripts/avisos` del worktree `feat/alertas` por ruta
  explícita en su prompt, sin tocar los symlinks, y se verifica en su sesión qué copia leyó.
- [ ] Anotar la evidencia en `intent/alertas/avance.md`, commitear y pedir OK para mergear a
  `main` y pushear.

## Tests a escribir
| Test | REQ |
| --- | --- |
| `test_avisos.py`: solo pasan bloques completos; `draft`, chat, token, `stream` y JSON roto no | REQ-3 |
| E33 resumen: aviso en `orq-export` esperando cierre, `orq-import` ilegible, pestañas de worker, del supervisor, de un chat común y de otro proyecto, dos altas en una pestaña, un alta cuya pestaña ya cerró, un posible traspaso | REQ-2, REQ-3, REQ-4, REQ-8 |
| E34 cierre: pregunta sin `pregunta:`; `1`, "sí, pero…" y "dale"; `run_not_found`; sin supervisor | REQ-1, REQ-5, REQ-8 |
| E35 Para cerrar: `orq-a`, `orq-b` con aviso nuevo, `orq-c` ilegible, cada una con un OK nuevo; `orq-c` sigue ilegible y una que falla recién en la segunda lectura; aviso ya mostrado; mensaje nuevo entre resumen y OK | REQ-3, REQ-6 |
| E36 traspaso: propuesta que pide mirar; un aviso charlado y otro no; la nueva muestra solo el charlado; sin avisos; sin supervisor | REQ-1, REQ-7, REQ-8 |
| E17 y E18 actualizados | REQ-5, REQ-6 |

## Riesgos
- El formato del mod puede cambiar y los avisos dejarían de verse sin error (aceptado en el spec).
- Aceptados: el aviso que aparece entre la relectura y el cierre; la pestaña nueva de un traspaso,
  sin leer hasta su próximo mensaje; ante un "no" a la pregunta de cierre, el orquestador la
  repite en cada turno mientras la pestaña siga abierta.
- Tocar los pasos 4, 5 y 6 y "Contexto" puede romper E3, E6, E12, E14 y E23; por eso se repiten.
