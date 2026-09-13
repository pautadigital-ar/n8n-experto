---
name: n8n-experto
description: "Forma de trabajo de Pauta Digital para construir, corregir, auditar u ordenar workflows de n8n: convenciones propias (nombres de nodos en español, notas, zona horaria y moneda argentinas), el patrón de manejo de errores que no deja fallas silenciosas, las trampas que ya costaron horas (triggers creados por API sin webhookId, un webhook por bot de Telegram, credenciales de Google que caducan a los 7 días, el MCP que borra en vez de archivar) y el checklist de cierre. Usala SIEMPRE que se trabaje con n8n o con el MCP de n8n en este entorno — crear un flujo, arreglar uno que 'no responde' o 'falla', conectar Telegram o Google Sheets, revisar ejecuciones, activar, limpiar o archivar workflows — aunque el usuario no la nombre. Se usa junto con el pack de skills n8n (using-n8n-mcp-skills), no en su lugar."
---

# n8n — forma de trabajo de Pauta Digital

Esta skill es una **capa propia** sobre el pack público `czlonkowski/n8n-skills`. Se distribuye junto a las skills originales en el fork `pautadigital-ar/n8n-experto`. El pack cubre lo genérico de n8n; esta skill cubre cómo se trabaja acá y lo que se aprendió a los golpes. No repite el pack: lo manda a usar.

Los datos concretos de la instancia (URL, IDs de credenciales, qué workflow hace qué) **no van acá**: viven en la memoria del proyecto. Esta skill es el *cómo*, reutilizable en cualquier instancia o cliente.

## 1. Antes de tocar nada

**Cargá las skills del pack.** Instaladas como copia manual (carpetas en `~/.claude/skills/`), no hay ningún hook que recuerde invocarlas: si no las cargás vos, no se cargan. Instaladas como plugin, el hook de inicio carga el índice, pero las especialistas igual hay que invocarlas. En la sesión que dio origen a esta skill se saltearon casi todas, y eso terminó en un parámetro mal pasado al MCP y en un manejo de errores que le mentía al usuario.

| Vas a… | Cargá |
|---|---|
| Cualquier cosa con n8n | `using-n8n-mcp-skills` (el índice) |
| Llamar herramientas del MCP | `n8n-mcp-tools-expert` |
| Diseñar un flujo nuevo | `n8n-workflow-patterns` |
| Configurar nodos | `n8n-node-configuration` |
| Escribir expresiones `{{ }}` | `n8n-expression-syntax` |
| Escribir un nodo Code | `n8n-code-javascript` |
| Armar ramas de error | `n8n-error-handling` |
| Manejar fotos, PDFs, archivos | `n8n-binary-and-data` |
| Entender un error de validación | `n8n-validation-expert` |

**Si el pack y la herramienta viva no coinciden, gana la herramienta.** Caso real: los ejemplos de `updateNode` en `n8n-error-handling` y otras skills del pack usan `changes`, pero el MCP lo rechaza y exige `updates` (la propia `n8n-mcp-tools-expert` sí dice `updates`). Avisale al usuario cuando encuentres una discrepancia así.

**Al crear nodos por API, usá un UUID v4 como `id`** (`[guid]::NewGuid()` en PowerShell), no nombres como `n1` o `http-node`. Lo indica `n8n-mcp-tools-expert`: n8n usa esos IDs en la interfaz y los legibles la rompen de formas sutiles.

**Mirá un workflow que ya funcione en la instancia antes de crear uno.** Te da tres cosas que no se adivinan: qué `typeVersion` de cada nodo acepta esa instancia, los IDs de credenciales (en algunas instancias listar credenciales por API devuelve 405, y leerlas de un workflow existente es la salida), y cómo están configurados los triggers que sí reciben eventos.

## 2. Convenciones

- **Nombres de nodo en español, con prefijo por tipo:** `Telegram - Confirmación`, `Code - Normalizar gastos`, `HTTP - OpenRouter extraer gasto`, `Sheets - Registrar gasto`. Se lee el flujo sin abrir cada nodo, y las expresiones `$('Nodo')` quedan autoexplicativas.
- **Nota (`notes`) en cada nodo que no sea obvio**, explicando el *porqué*, no el qué: por qué `executeOnce`, por qué esa credencial, qué hay que revisar si falla. Es lo que va a leer el usuario (o Claude) dentro de seis meses.
- **Sticky notes** con la guía de configuración y todo lo que queda pendiente de parte del usuario. Cuando algo se completa, se actualiza la nota: una nota vieja que dice "falta configurar" confunde.
- **Settings del workflow:** `executionOrder: "v1"`, `timezone: "America/Argentina/Buenos_Aires"`, y guardar ejecuciones exitosas y fallidas (sin eso no hay forma de diagnosticar después).
- **Datos:** fechas `YYYY-MM-DD`, mes como `YYYY-MM` (ordena bien como texto), montos como número puro sin símbolo. Moneda por defecto `ARS`. Al parsear números argentinos, el punto separa miles y la coma decimales: `1.234,56` → `1234.56`.
- **Textos que ve el usuario final** (mensajes de Telegram, avisos): en español rioplatense y sin Markdown salvo que se configure `parse_mode`, porque si no aparecen los asteriscos literales. En los nodos de Telegram que envían mensajes, `additionalFields.appendAttribution: false`: por defecto n8n agrega "This message was sent automatically with n8n" al final.
- **Si el usuario puede escribir una nota** junto a una foto o archivo (epígrafe), el prompt de la IA tiene que decir que la nota manda sobre lo que se ve. Si no, la IA la trata como un comentario más y la ignora.
- **Secretos siempre en el sistema de credenciales de n8n**, nunca en parámetros ni en nodos Set.

## 3. Manejo de errores: nada de fallas silenciosas

Una salida de error que avisa y sigue hace que la ejecución figure como **"success"** aunque el trabajo se haya perdido. El pack lo advierte: una salida de error atrapada cuenta como "manejada" y además suprime el error workflow. En la práctica eso significó dos días de gastos perdidos sin ninguna señal en n8n.

El patrón:

1. `onError: "continueErrorOutput"` en cada nodo que puede fallar (HTTP, Code, Sheets, APIs) y su salida de error cableada.
2. Un nodo de aviso que diga **qué nodo falló** y **qué hacer**, sin asumir la causa. Un aviso que dice "mandá una foto más nítida" cuando el problema es una credencial vencida manda al usuario a buscar en el lugar equivocado.
3. Después del aviso, un nodo **`Stop and Error`** para que la ejecución quede marcada como fallida y dispare el error workflow si hay uno.

Detalle, ejemplos de configuración y cómo verificarlo: `references/errores.md`.

## 4. Trampas conocidas

| Síntoma | Causa | Detalle |
|---|---|---|
| El workflow activa sin error pero **nunca recibe eventos** (0 ejecuciones) | Trigger de webhook creado por API **sin `webhookId`** | `references/trampas.md` |
| Un bot de Telegram que andaba **dejó de recibir mensajes** | Otro workflow activó un trigger con el **mismo bot**: Telegram admite un solo webhook por bot | `references/trampas.md` |
| Google Sheets/Drive falla con *"authorization grant … invalid, expired, revoked"* después de unos días | App OAuth de Google en estado **"Prueba"**: el refresh token caduca a los 7 días | `references/google-sheets.md` |
| Ejecuciones en "success" pero el dato **no llegó** | La rama de error lo atrapó | sección 3 |
| "Test workflow" en el editor tira un error de JavaScript y no crea ejecución | Problema del editor, no necesariamente del flujo | `references/trampas.md` |
| Hay que **archivar** workflows | El MCP no archiva: `n8n_delete_workflow` **borra para siempre** | `references/archivar.md` |

**Método que destrabó casi todo:** ante un flujo que "no anda", compará contra uno que sí ande en la misma instancia (mismo proveedor, misma infraestructura). Si el otro recibe eventos y este no, el problema es del nodo, no de la red ni de n8n. Y leé las ejecuciones con `n8n_executions` antes de adivinar: el preview muestra qué nodo devolvió `error` aunque la ejecución figure como exitosa.

## 5. Acciones delicadas

- **Nunca uses `n8n_delete_workflow` para "archivar" o "limpiar".** No se puede deshacer.
- **Operaciones en lote** (archivar, desactivar, cambiar credenciales en muchos workflows): probá primero con uno solo y confirmá el resultado, poné una guarda que excluya explícitamente los workflows activos, y verificá al final por otra vía (si ejecutaste desde el navegador, verificá con el MCP).
- **Activar** un workflow con trigger externo registra el webhook en el proveedor y puede pisar otro: revisá antes que ningún workflow activo use la misma credencial de trigger.
- **Ejecutar con efectos reales** (manda mensajes, escribe planillas, cobra APIs): avisá antes.
- **Credenciales y claves privadas** (tokens de bots, JSON de cuentas de servicio): las carga el usuario en n8n. No se pegan en el chat ni se manipulan desde el navegador.

## 6. Checklist de cierre

No des un flujo por terminado sin esto:

- [ ] Cargaste las skills del pack que correspondían (sección 1).
- [ ] `n8n_validate_workflow` sin errores, y las advertencias revisadas una por una.
- [ ] `n8n_get_workflow` en modo `structure`: cada conexión va a donde tiene que ir, incluidas las salidas de error.
- [ ] Todo trigger de webhook tiene `webhookId`.
- [ ] Ningún otro workflow activo usa la misma credencial de trigger (Telegram).
- [ ] Nodos que fallan → aviso con el nodo que falló → `Stop and Error`.
- [ ] `typeVersion` compatibles con la instancia (compará con un workflow que funcione).
- [ ] Notas y sticky notes actualizadas, sin pendientes que ya no aplican.
- [ ] Probado de punta a punta con un caso real, y **leída la ejecución** en `n8n_executions`: no alcanza con que "no dio error".
- [ ] Le dijiste al usuario, sin adornos, qué quedó pendiente de su lado.
