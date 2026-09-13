# n8n-experto

Repertorio de skills de **Pauta Digital** para construir, corregir y mantener workflows de n8n con Claude Code a través del MCP de n8n.

Reúne el pack público [czlonkowski/n8n-skills](https://github.com/czlonkowski/n8n-skills) completo y, encima, **`n8n-experto`**: una skill propia con nuestras convenciones de trabajo, el patrón de manejo de errores que usamos, las trampas que ya nos costaron horas y el checklist de cierre de cada flujo.

> Este repo es un fork. Las skills originales no se modifican: se mantienen sincronizadas con el autor. Lo propio vive en `skills/n8n-experto/`.

---

## Qué incluye

### Skill propia

| Skill | Para qué |
|---|---|
| **`n8n-experto`** | Convenciones (nodos en español, notas, zona horaria y moneda argentinas), errores que no quedan en silencio (aviso con el nodo que falló + `Stop and Error`), trampas conocidas y checklist de cierre. Se usa junto con el pack, no en su lugar. |

Referencias en `skills/n8n-experto/references/`:
- `trampas.md`: triggers creados por API sin `webhookId`, un webhook por bot de Telegram, ejecuciones "success" que perdieron datos y más.
- `errores.md`: patrón de manejo de errores, verificado en producción.
- `google-sheets.md`: cuenta de servicio en vez de OAuth, para que las credenciales no caduquen a los 7 días.
- `archivar.md`: archivar workflows en lote sin borrarlos.

### Skills del pack original

| Skill | Para qué |
|---|---|
| `using-n8n-mcp-skills` | Índice: indica qué skill usar en cada tarea |
| `n8n-mcp-tools-expert` | Uso correcto de las herramientas del MCP de n8n |
| `n8n-workflow-patterns` | Arquitectura de workflows |
| `n8n-node-configuration` | Configuración de nodos según la operación |
| `n8n-expression-syntax` | Expresiones `{{ }}` |
| `n8n-code-javascript` / `n8n-code-python` | Nodos Code |
| `n8n-code-tool` | Custom Code Tool para agentes de IA |
| `n8n-error-handling` | Manejo de errores y reintentos |
| `n8n-binary-and-data` | Archivos, imágenes y datos binarios |
| `n8n-validation-expert` | Interpretar errores de validación |
| `n8n-subworkflows` | Sub-workflows reutilizables |
| `n8n-agents` | Agentes de IA |
| `n8n-multi-instance` | Varias instancias de n8n |
| `n8n-self-hosting` | Instalar n8n en un servidor propio |

---

## Instalación

Desde una terminal interactiva de Claude Code:

```
/plugin marketplace add pautadigital-ar/n8n-experto
/plugin install n8n-experto@n8n-experto
```

Si antes copiaste las skills a mano en `~/.claude/skills/`, borrá esas carpetas para no tener duplicados.

Necesitás además el servidor MCP de n8n: [czlonkowski/n8n-mcp](https://github.com/czlonkowski/n8n-mcp).

---

## Actualizaciones

Las skills dependen de versiones concretas del MCP y de n8n, y cambian seguido. Dónde mirar:

| Qué | Dónde |
|---|---|
| Nuevas versiones del pack de skills | [czlonkowski/n8n-skills/releases](https://github.com/czlonkowski/n8n-skills/releases) |
| Nuevas versiones del MCP de n8n | [czlonkowski/n8n-mcp/releases](https://github.com/czlonkowski/n8n-mcp/releases) y su [CHANGELOG](https://github.com/czlonkowski/n8n-mcp/blob/main/CHANGELOG.md) |
| Notas de versión de n8n | [docs.n8n.io/changelog/release-notes](https://docs.n8n.io/changelog/release-notes) |

**Para traer las mejoras del autor:** en la página del repo, **Sync fork → Update branch**. Si hay un conflicto, casi seguro va a estar en `.claude-plugin/marketplace.json`: conservá la línea `"./skills/n8n-experto"` y el nombre `n8n-experto`.

Cuando cambie algo en el MCP o en n8n que afecte lo que dice `n8n-experto`, se actualiza la skill propia.

---

## Créditos y licencia

Las skills originales son obra de **Romuald Członkowski** ([czlonkowski/n8n-skills](https://github.com/czlonkowski/n8n-skills)), publicadas bajo licencia MIT. La licencia y su aviso de copyright se mantienen en [`LICENSE`](../LICENSE).

La skill `n8n-experto` y este README son de Pauta Digital.
