# Spec: supervisar (desde intent.md 2026-11-12)
Estado: aceptado

## Requisitos
- REQ-1 MUST: Un orquestador con supervisor se da de alta con un solo mensaje: intent, proyecto,
  rama y su dirección. Origen: Resultado esperado, "le van diciendo todo".
- REQ-2 MUST: Cada orquestador tiene una dirección propia que lo despierta al recibir un mensaje
  y que no cambia con sus traspasos. Origen: Resultado esperado, "Puedo responder desde el
  supervisor o directamente en la pestaña".
- REQ-3 MUST: Una pregunta que necesita decisión del usuario se hace en la pestaña y se manda al
  supervisor. Se responde en cualquiera de los dos: la respuesta llega al orquestador, y la dada
  en la pestaña llega al supervisor citando la pregunta. Origen: Resultado esperado,
  "en los dos casos la decisión le llega al supervisor".
- REQ-4 MUST: Los orquestadores mandan solo alta, pregunta, decisión y cierre (terminó o
  abandonó); no reportan progreso. Origen: Restricciones, "alcanza con mensajes, checkpoints y
  resúmenes".
- REQ-5 MUST: A pedido, el supervisor resume: orquestadores vivos y su intent, preguntas
  abiertas, bloqueos y duplicados (mismo intent, o mismo proyecto y rama). Antes de dar una
  pregunta por abierta, verifica que no se haya respondido en la pestaña. Origen: Resultado
  esperado, "me da un resumen".
- REQ-6 MUST: Ante una idea nueva, el supervisor dice si va a un intent existente o a uno nuevo y
  propone el paso siguiente; no lanza, cierra ni redirige a nadie sin aprobación. Origen:
  Resultado esperado, "propone acciones y pide aprobación".
- REQ-7 MUST: El supervisor no lleva estado propio: lo reconstruye de los mensajes, `intent/` y
  las sesiones vivas, y al hacer traspaso conserva su dirección. Origen: Resultado esperado,
  "conserva la dirección".
- REQ-8 MUST: Tras el traspaso de un orquestador, la pestaña vieja se cierra cuando la nueva
  confirma que tomó el trabajo, y el supervisor no recibe mensajes por eso. Origen: Resultado
  esperado, "la pestaña anterior no queda viva"; Restricciones, "Hago handoff cerca del 30 % en el supervisor y del 40 % en un orquestador".
- REQ-9 MUST: Sin supervisor activo, el orquestador trabaja como hoy y no crea uno. Hay uno por
  proyecto: un repo con todos sus worktrees. Si encuentra más de uno vivo en el mismo proyecto, le
  pregunta al usuario. Origen: Resultado esperado, "no es el único canal"; Cambios 2026-11-13.
- REQ-10 MUST: El traspaso actual sigue igual sin supervisor y solo suma su dirección cuando hay
  uno. Origen: Restricciones, "sin texto repetido entre skills".
- REQ-11 SHOULD: Al pasar su umbral de contexto (supervisor 30 %, orquestador 40 %), el agente
  propone el traspaso y espera aprobación. Origen: Restricciones, "Hago handoff cerca del 30 %
  en el supervisor y del 40 % en un orquestador".
- REQ-12 MUST: Al terminar o abandonar un intent, el orquestador cierra los workers que siguen
  abiertos en su run y propone borrar los worktrees que creó y ya están mergeados (los borra con
  OK); si el cierre sale sin `run_not_found`, cierra su pestaña como última acción y, si no, le
  muestra el resultado al usuario. Origen: Resultado
  esperado, "la pestaña anterior no queda viva" (agregado 2026-11-18, aprobado por Denis).
- REQ-13 MUST: El resumen del supervisor lista lo que sobra (pestañas de orquestadores cerrados
  que siguen abiertas y worktrees de intents terminados) y, con un solo OK del usuario, lo cierra;
  si algo no se puede cerrar, sigue con el resto y lo informa.
  Origen: Resultado esperado, "propone acciones y pide aprobación" (agregado 2026-11-18,
  aprobado por Denis).
- REQ-14 SHOULD: Al arrancar, de cero o por traspaso, el supervisor nombra su pestaña de Orca
  `sup-<repo>`, para que el usuario filtre los supervisores por ese prefijo. Origen: Problema, "me
  termino volviendo loco saltando entre chat y chat" (agregado 2026-11-24, aprobado por Denis).
- REQ-15 SHOULD: Con supervisor, el orquestador nombra su pestaña de Orca `orq-<slug>` (la ruta
  del intent entre `intent/` y `/intent.md`, con `-` en vez de `/`; `orq-<repo>` para el intent
  raíz) después de que su alta salga sin `run_not_found`, o al retomar su run en un traspaso. Sin
  supervisor no la renombra. Origen: Problema, "Por cada intent o idea termino con 4 o 5 chats
  tipo orquestador" (agregado 2026-11-24, aprobado por Denis).

## Capacidades y escenarios
### Alta y traspaso (REQ-1, REQ-2, REQ-8)
- GIVEN un supervisor activo
- WHEN arranca un orquestador y, más tarde, cerca del 40 % de contexto, hace traspaso
- THEN el supervisor recibe un único alta; la pestaña nueva toma la misma dirección y la vieja se
  cierra

### Pregunta respondida en la pestaña (REQ-3, REQ-5)
- GIVEN una pregunta mandada al supervisor
- WHEN el usuario la responde en la pestaña del orquestador
- THEN el supervisor recibe la decisión y no la muestra como abierta

### Idea nueva (REQ-6)
- GIVEN un intent abierto sobre el mismo tema
- WHEN el usuario le cuenta la idea al supervisor
- THEN el supervisor propone sumarla a ese intent y espera aprobación

### Supervisor sin contexto (REQ-7)
- GIVEN un supervisor haciendo traspaso
- WHEN un orquestador le escribe durante el cambio
- THEN el supervisor nuevo recibe el mensaje

### Umbral de contexto (REQ-11)
- GIVEN un supervisor al 31 % de contexto
- WHEN termina de atender un mensaje
- THEN propone el traspaso y no lo hace hasta que el usuario acepte

### Sin supervisor (REQ-9, REQ-10)
- GIVEN ningún supervisor activo
- WHEN un orquestador hace traspaso
- THEN todo es igual que hoy

### Cierre (REQ-12, REQ-13)
- GIVEN un orquestador que termina su intent con un worker abierto y un worktree mergeado
- WHEN manda el cierre
- THEN antes cerró el worker y preguntó por el worktree, y después cerró su pestaña; si algo
  quedó abierto, el siguiente resumen del supervisor lo lista en "Para cerrar" y lo cierra solo
  con el OK del usuario

### Pestaña del supervisor (REQ-14)
- GIVEN el repo `foo`
- WHEN arranca un supervisor, de cero o por traspaso
- THEN su pestaña de Orca se llama `sup-foo`

### Pestaña del orquestador (REQ-15)
- GIVEN un supervisor activo y el intent `intent/export/intent.md`
- WHEN el orquestador manda su alta y sale sin `run_not_found`, o retoma su run en un traspaso
- THEN su pestaña de Orca se llama `orq-export`

## Concerns
- Sin probar: que la pestaña nueva tome la dirección de un orquestador vivo sin forzar; se
  verifica en el plan.
- Probado el 2026-11-12: `ask` solo funciona en workers; las preguntas van como mensaje con
  respuesta.
- Cada mensaje cuesta un turno del supervisor, que es el que más rápido se llena: de ahí REQ-4,
  REQ-7 y REQ-8.
- Una decisión dada en la pestaña y no avisada: REQ-5 lo mitiga, no lo garantiza.
- Un orquestador que muere sin cierre se detecta cruzando altas con sesiones vivas.
- Abierto: cómo cargan la skill Codex y Antigravity.
- Sin verificar: cuánto guarda Orca los mensajes.
