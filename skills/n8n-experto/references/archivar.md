# Archivar workflows

## El MCP no archiva

`n8n_update_partial_workflow` no tiene operación de archivado, y `n8n_delete_workflow` **borra para siempre** ("This action cannot be undone"). Nunca lo uses cuando el usuario dice "archivar", "limpiar" o "sacar del medio".

Archivar sí es reversible: los workflows archivados siguen existiendo y se pueden restaurar desde la interfaz de n8n.

## Opciones

- **Pocos workflows:** interfaz de n8n → menú `⋮` de cada fila → **Archive**. No hay selección múltiple.
- **Muchos:** la API interna de n8n desde el navegador, con la sesión ya iniciada del usuario. Es el mismo endpoint que usa el botón. Pedile confirmación al usuario del alcance y del método antes de correrlo.

## Procedimiento en lote

1. **Listá con el MCP:** `n8n_list_workflows` con `active: false`, y filtrá `isArchived: false`. Mostrale al usuario el conteo y separá lo que es trabajo de clientes de lo que es basura evidente (tests, copias, "My workflow N", 0 nodos): "viejos" es ambiguo.
2. **Anotá los IDs activos** (`active: true`) como protegidos.
3. **Probá con uno solo**, el más descartable, desde la pestaña de n8n con `javascript_tool`:

```javascript
const bid = localStorage.getItem('n8n-browserId') || '';
const r = await fetch('/rest/workflows/<ID>/archive', {
  method: 'POST', credentials: 'include',
  headers: { 'Content-Type': 'application/json', 'browser-id': bid }
});
const j = await r.json();
({ status: r.status, archivado: j?.data?.isArchived, activo: j?.data?.active, nombre: j?.data?.name })
```

Tiene que devolver `status: 200` y `archivado: true`. Es una API interna: si cambia la ruta o devuelve 401/404, frená y avisá en vez de probar variantes a ciegas.

4. **Corré el resto** con guarda y reporte:

```javascript
const PROTEGIDOS = [/* IDs activos */];
const ids = [/* IDs a archivar */];
const bid = localStorage.getItem('n8n-browserId') || '';
const ok = [], fallaron = [];
for (const id of ids) {
  if (PROTEGIDOS.includes(id)) continue;
  try {
    const r = await fetch('/rest/workflows/' + id + '/archive', {
      method: 'POST', credentials: 'include',
      headers: { 'Content-Type': 'application/json', 'browser-id': bid }
    });
    const w = (await r.json().catch(() => ({})))?.data;
    if (r.ok && w?.isArchived === true && w.active !== true) ok.push(w.name);
    else fallaron.push({ id, status: r.status, activo: w?.active });
  } catch (e) { fallaron.push({ id, motivo: String(e).slice(0, 120) }); }
  await new Promise(res => setTimeout(res, 120));
}
({ intentados: ids.length, archivados: ok.length, fallaron, nombres: ok })
```

5. **Verificá por otra vía:** `n8n_list_workflows` con el MCP. Tienen que quedar sin archivar solamente los activos.
