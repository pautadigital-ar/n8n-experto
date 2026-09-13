# n8n-experto

Skill propia de Pauta Digital para trabajar con n8n: convenciones, patrón de manejo de errores, trampas conocidas y checklist de cierre.

Es una capa sobre el pack público [czlonkowski/n8n-skills](https://github.com/czlonkowski/n8n-skills) (licencia MIT). Vive en el fork [pautadigital-ar/n8n-experto](https://github.com/pautadigital-ar/n8n-experto) junto a las skills originales, que no se modifican. No copia su contenido: las referencia por nombre y manda a cargarlas.

## Estructura

- `SKILL.md` — qué skills del pack cargar, convenciones, errores, trampas, acciones delicadas y checklist.
- `references/trampas.md` — cada trampa con síntoma, confirmación y arreglo.
- `references/errores.md` — patrón aviso + Stop and Error.
- `references/google-sheets.md` — cuenta de servicio en vez de OAuth.
- `references/archivar.md` — archivado en lote sin borrar.

Los datos de cada instancia (URLs, IDs de credenciales) no van acá: viven en la memoria del proyecto.
