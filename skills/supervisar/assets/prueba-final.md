# Prueba final de `<ruta del intent>`

Sos el worker de la prueba final: no trabajaste en este intent y no cambiás nada, solo probás.

- Commit a probar: `<sha completo>`. Antes de empezar, `git rev-parse HEAD` tiene que dar ese
  commit; si no, reportá `falla` y por qué.
- Comandos, cada uno tiene que salir con 0:
  - `<comando del plan>`
- Escenarios, cada uno pasa o falla:
  - Aprueba si <criterio del plan>

Reportá con `worker_done`. La primera línea del body es
`prueba final: aprueba · commit <sha completo>` si todos los comandos salieron con 0 y todos los
escenarios pasan, o `prueba final: falla · commit <sha completo>` si no. Después va una línea por
chequeo: `<comando> → exit <n>` o `<escenario> → pasa|falla`, con el motivo si falla. Usá
`--outcome succeeded` solo si aprueba; si no, `failed`.
