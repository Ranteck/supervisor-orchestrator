# Intención: alertas
Autor: Denis Hugo Perafan · Estado: aceptado · Fecha: 2026-11-24

## Problema
"En varios chats se corren side agents, en donde van avisando errores. Eso sí es importante."
Son modelos que van corriendo, buscando bugs hasta que los encuentran y te avisan, y le preguntan
al usuario si desea charlarlo en el chat o no. Hoy no sé cómo hacer para que el supervisor lo vea
o se entere, sin que yo vaya chat por chat revisando si no lo tiene.

## Resultado esperado
El supervisor me muestra en el resumen los avisos que haya en las pestañas de sus orquestadores,
con su origen, o me dice qué pestaña no pudo leer. Antes de que se cierre una pestaña de
orquestador, tengo oportunidad de ver sus avisos:
- al terminar, el orquestador me pregunta y espera mi OK antes de cerrar su pestaña: así reviso, y
  ahí puede ser que aparezca;
- en "Para cerrar", el supervisor vuelve a mirar y me muestra los avisos nuevos;
- en un traspaso, me pide mirarlos antes de aprobarlo, y los que charlo pasan a la pestaña nueva,
  que me los muestra al arrancar.

## Usuarios y sistemas afectados
Yo; supervisor y orquestadores de `supervisar`; los avisos "Heads up" del mod "You should know" de
Claude Code.

## Restricciones
Probado (2026-11-24/25):
- El aviso aparece al terminar un turno y solo en pantalla: no queda en el `.jsonl` ni en
  `orca search`; `orca terminal read` lo lee mientras se vea.
- Si lo charlo, entra como turno de usuario; recién ahí lo ve el orquestador.
- De la pantalla se toman solo las líneas del aviso: el resto puede tener secretos.
- Cambia el cierre aceptado (REQ-3, REQ-8, REQ-12 y REQ-13 del spec raíz), sin mensajes nuevos.

## Fuera de alcance
- Que el supervisor responda o descarte un aviso, o escriba en otra pestaña.
- Reenviar avisos al supervisor por mensajes de Orca.
- Leer el archivo interno del mod: no dice de qué pestaña es cada nota.
- Pestañas de workers, del supervisor, chats comunes y otros proyectos: el patrón es supervisor →
  orquestador → worker.
- Avisos que se pierden antes de leerlos (un turno nuevo los borra, o aparecen después): se acepta
  ese límite.
- Cambiar Claude Code o el mod.

## Preguntas abiertas
- ¿El aviso se borra solo con un turno nuevo, o también con el tiempo? Prueba en curso.

## Cambios
2026-11-25: en un traspaso pasan solo los avisos que charlo, porque el turno que lo aprueba borra
el aviso y la pestaña vieja ya no lo puede leer. Aprobado por Denis.
