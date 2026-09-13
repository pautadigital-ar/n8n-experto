# Trampas conocidas

Cada una con cómo se ve, cómo se confirma y cómo se arregla. Todas pasaron de verdad.

## 1. Trigger de webhook sin `webhookId`

**Cómo se ve:** el workflow activa con `success: true`, figura como activo, y nunca recibe nada. `n8n_executions` devuelve 0 ejecuciones. Otros workflows con el mismo proveedor funcionan normal.

**Por qué pasa:** n8n arma la URL de producción como `<WEBHOOK_URL>/webhook/<webhookId>`. Cuando arrastrás el nodo en la interfaz, n8n genera ese UUID solo; cuando creás el workflow por API (`n8n_create_workflow`), no. Sin el campo, la activación registra una ruta inválida y el proveedor manda los eventos a ningún lado. Afecta a Telegram Trigger, Webhook y cualquier nodo trigger basado en webhook. El pack de skills no lo menciona.

**Cómo se confirma:** `n8n_get_workflow` en modo `filtered` con el nombre del trigger. Comparalo con un trigger que funcione: el que anda tiene `"webhookId": "<uuid>"` y el roto no.

**Cómo se arregla:** en este orden, porque cambiar el id con el workflow activo puede dejar un registro viejo colgado:

1. `deactivateWorkflow`
2. `updateNode` con `updates: { "webhookId": "<uuid v4 nuevo>" }`
3. `activateWorkflow` (vuelve a registrar el webhook)
4. Verificá el campo con `n8n_get_workflow` y probá con un evento real.

**Prevención:** al crear por API, incluí siempre `webhookId` en el nodo trigger. Un UUID se genera en PowerShell con `[guid]::NewGuid().ToString()`.

## 2. Telegram: un solo webhook por bot

**Cómo se ve:** un workflow con Telegram Trigger que funcionaba deja de recibir mensajes, justo después de activar otro workflow.

**Por qué pasa:** `setWebhook` de Telegram pisa el webhook anterior del bot. Dos workflows activos con el mismo bot → solo el último activado recibe.

**Cómo se arregla:** un bot por flujo. El usuario crea el bot con @BotFather y carga su propia credencial "Telegram API" en n8n. La alternativa (un solo workflow con un router que separe los casos) toca un flujo en producción: ofrecela como opción, no la asumas.

**Prevención:** antes de activar, revisá qué credencial de Telegram usan los workflows activos.

## 3. Credenciales de Google que caducan

Ver `google-sheets.md`.

## 4. "Test workflow" tira un error de JavaScript

**Cómo se ve:** al apretar "Test workflow" en el editor aparece *"Problem running workflow — Cannot read properties of undefined (reading '…')"*, y no se crea ninguna ejecución.

**Qué se sabe:** que no se cree ejecución indica que falla antes de arrancar, del lado del editor. En el caso real nunca se confirmó la causa exacta: se bajaron los `typeVersion` de dos nodos por precaución, y el flujo terminó funcionando al activarlo y arreglar el `webhookId` (trampa 1).

**Sospecha principal, sin confirmar:** ese workflow se había creado por API con IDs de nodo legibles (`n1`, `n2`…). `n8n-mcp-tools-expert` advierte que n8n usa los IDs de nodo en la interfaz y que los que no son UUID causan fallas sutiles. Encaja con un error que lee una propiedad de algo `undefined` en el editor. Para confirmarlo habría que recrear el workflow con UUIDs y ver si "Test workflow" deja de fallar.

**Qué hacer:** no persigas el error del editor como primera medida.

1. Validá y revisá las conexiones por MCP.
2. Compará los `typeVersion` contra un workflow que funcione en la instancia; usá los que ya estén probados ahí.
3. Activá por API: si hay un problema estructural real, la activación lo reporta con un mensaje del servidor.
4. Revisá el `webhookId` del trigger.

No le digas al usuario que el error del editor quedó resuelto si no lo confirmaste.

## 5. Ejecuciones "success" que perdieron el dato

**Cómo se ve:** el usuario dice que algo no se guardó, pero todas las ejecuciones figuran en verde.

**Por qué pasa:** la rama de error atrapó la falla, mandó un aviso y la ejecución terminó bien.

**Cómo se confirma:** `n8n_executions` con `mode: "preview"` muestra en cada nodo si su salida tiene `error`. Para leer el mensaje exacto, `mode: "filtered"` con `nodeNames` del nodo sospechoso. Para ubicar cuándo empezó, abrí ejecuciones de fechas distintas y buscá la última buena.

**Cómo se arregla:** el patrón de `errores.md`.

## 6. Parámetros del MCP que no coinciden con la documentación

- `updateNode` exige `updates`, no `changes` (el ejemplo del pack está desactualizado).
- `get_node` y `validate_node` usan tipo corto (`nodes-base.telegram`); el JSON del workflow usa tipo largo (`n8n-nodes-base.telegram`).
- Para cambiar la autenticación de un nodo de Google Sheets a cuenta de servicio: `parameters.authentication: "serviceAccount"` y reemplazar el objeto `credentials` completo por `{ "googleApi": { "id": "…", "name": "…" } }`, así no queda la credencial OAuth colgada.

Ante la duda, `tools_documentation` con el nombre de la herramienta y `depth: "full"`.
