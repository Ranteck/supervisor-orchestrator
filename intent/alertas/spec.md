# Spec: alertas (desde intent.md 2026-11-24)
Estado: aceptado

`RE`: el párrafo de "Resultado esperado"; `RE n`: su viñeta n. "Aviso": la línea `✦`
del mod "You should know" bajo el prompt.

## Requisitos
- REQ-1 MUST: Rige solo con supervisor; sin él, el orquestador trabaja como hoy. Origen: Usuarios.
- REQ-2 MUST: Al pedir el resumen, el supervisor lee cada pestaña abierta de sus orquestadores,
  también las que esperan cierre, y muestra cada aviso con su pestaña e intent, o la que no pudo
  leer. Origen: RE.
- REQ-3 MUST: De toda lectura de pantalla solo pasan las líneas del aviso. Origen: Restricciones.
- REQ-4 MUST: No lee pestañas de workers, la propia, chats comunes ni de otros proyectos. Origen:
  Fuera de alcance.
- REQ-5 MUST: Si el `cierre` salió sin `run_not_found`, el orquestador pregunta solo en su pestaña
  si la cierra, sin `pregunta:`, y la cierra solo con un OK inequívoco; ante otra respuesta la deja
  abierta y vuelve a preguntar después. Origen: RE 1.
- REQ-6 MUST: En "Para cerrar", el supervisor relee cada pestaña aprobada antes de cerrarla; si
  muestra un aviso que no figuraba en el resumen o no se pudo leer, la deja abierta, lo informa y pregunta, y
  sigue con el resto. Origen: RE 2.
- REQ-7 MUST: En un traspaso, los avisos visibles de la pestaña vieja pasan a la nueva, que los
  muestra al arrancar con su origen, sin imitar el formato del aviso, o dice que no pudo leerla.
  Origen: RE 3.
- REQ-8 MUST: Sin mensajes nuevos ni lectura del archivo del mod; nadie responde un aviso por el
  usuario ni el supervisor escribe en otra pestaña. Origen: Restricciones; Fuera de alcance.

## Escenarios
### Resumen (REQ-2, REQ-3, REQ-4, REQ-8)
- GIVEN "✦ Heads up · Falta migrar la tabla" en `orq-export`, que espera su OK de cierre, WHEN se
  pide el resumen, THEN figura con `orq-export` e `intent/export/` y nada más de esa pantalla; el
  supervisor no actúa sobre el aviso.
- GIVEN `orq-import` ilegible, WHEN se arma el resumen, THEN dice que no se leyó.
- GIVEN avisos en pestañas de un worker y del supervisor, WHEN se arma, THEN no se leen.

### Cierre del orquestador (REQ-5)
- GIVEN un `cierre` sin `run_not_found`, WHEN el orquestador termina, THEN pregunta y cierra la
  pestaña solo con el OK; con `run_not_found`, no la cierra.
- GIVEN la pregunta de cierre, WHEN el usuario charla un aviso o responde otra cosa, THEN la
  pestaña sigue abierta y vuelve a preguntar.
- GIVEN que nadie responde, WHEN se pide el resumen, THEN la pestaña figura en "Para cerrar".

### Para cerrar (REQ-6)
- GIVEN un OK para `orq-a`, `orq-b` (aviso posterior al resumen) y `orq-c` (ilegible), WHEN el supervisor
  ejecuta, THEN cierra `orq-a`, deja las otras, muestra el aviso y dice que `orq-c` no se leyó;
  con otro OK cierra `orq-b`, con un no la deja.
- GIVEN un aviso ya mostrado en el resumen, WHEN ejecuta, THEN la cierra.

### Traspaso (REQ-3, REQ-7)
- GIVEN la pestaña vieja con un aviso, WHEN hace traspaso, THEN la nueva lo muestra al arrancar
  con su origen y nada más de la vieja, que se cierra como hoy; si no se pudo leer, lo dice.

### Sin supervisor (REQ-1)
- GIVEN ningún supervisor, WHEN un orquestador cierra o traspasa, THEN nada cambia.

## Concerns
- Enmienda REQ-3 raíz (REQ-5 no manda `pregunta:`: una respuesta por Orca abriría un turno que
  borra el aviso, y "Para cerrar" cubre al que no responde), REQ-8, REQ-12 y REQ-13.
- Límite aceptado: un turno nuevo borra el aviso (dos casos, 2026-11-25); se pierde el que se
  borra o aparece después de leer. Sin probar si se borra con el tiempo.
- Riesgos aceptados: cambio de formato del mod (dejaría de verse sin error), texto del aviso
  sensible, pestaña nueva de traspaso sin leer hasta su próximo mensaje.
