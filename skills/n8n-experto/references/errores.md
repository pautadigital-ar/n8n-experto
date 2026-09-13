# Patrón de manejo de errores

Para los fundamentos (cableado de salidas de error, reintentos, error workflows), cargá `n8n-error-handling` del pack. Acá está cómo lo aplicamos.

## Por qué no alcanza con un aviso

Si una rama de error manda un mensaje y termina, n8n marca la ejecución como exitosa. Consecuencias:

- En la interfaz no podés filtrar por ejecuciones fallidas: no hay ninguna.
- El error workflow de la instancia (si existe) no se dispara, porque la falla quedó "manejada".
- Si el aviso se pierde o nadie lo lee, no hay ninguna otra señal.

Por eso el aviso va seguido de un `Stop and Error`.

## Forma del flujo

```
Nodo que puede fallar ──(salida 0)──▶ sigue el flujo
        │
   (salida 1: error)
        ▼
Aviso de error ──▶ Stop and Error
```

Varios nodos pueden compartir el mismo aviso: cableá todas sus salidas de error al mismo nodo.

## Configuración

**En cada nodo que puede fallar:**

```json
{ "onError": "continueErrorOutput", "retryOnFail": true, "maxTries": 2, "waitBetweenTries": 3000 }
```

Reintentos solo donde tenga sentido (APIs externas, Google). Un nodo Code que falla por datos malos no mejora reintentando. Límites del motor según `n8n-error-handling`: `maxTries` hasta 5 y `waitBetweenTries` hasta 5000 ms, y el reintento se dispara ante cualquier error (no se puede filtrar por código HTTP).

**Asignar un error workflow no se puede por MCP:** se hace en la interfaz, en la configuración del workflow → Error Workflow. Si armás uno, dejale al usuario ese paso indicado.

**Aviso de error** (ejemplo con Telegram; `executeOnce: true` para no mandar un mensaje por ítem):

```
={{ '⚠️ No se pudo completar.\n\nFalló: ' + $prevNode.name + '\nMotivo: ' + ($json.error?.message || $json.error || 'desconocido') }}
```

`$prevNode.name` da el nodo que mandó el ítem por su salida de error, **aunque el aviso reciba de varios nodos**. No está documentado en el pack, pero quedó **verificado en producción** el 2026-09-13: un aviso compartido por 4 nodos nombró correctamente `Code - Normalizar gastos` como el que falló. Si en otra versión de n8n viniera vacío, la alternativa es un nodo Set por rama que agregue el nombre antes del aviso.

Ojo con los errores de nodos Code: n8n les agrega la línea al final (`… en el mensaje. [line 31]`). Para mostrárselo al usuario, limpialo con `.replace(/\s*\[line \d+\]\s*$/, '')`.

Adaptá el "qué hacer" al tipo de nodo en vez de dar un consejo único:

- Falló la IA / el OCR → "probá con una foto más nítida o escribilo a mano".
- Falló Google Sheets → "problema de acceso a la planilla; revisá la credencial y que esté compartida".
- Falló una API externa → "servicio caído o credencial vencida; reintentá en unos minutos".

**Stop and Error** (`n8n-nodes-base.stopAndError`, `typeVersion: 1`):

```json
{
  "errorType": "errorMessage",
  "errorMessage": "Falló el flujo — ver el aviso enviado y la ejecución"
}
```

Lo importante es que la ejecución quede en rojo. Si querés un mensaje más rico, referenciá el campo que exponga el nodo de aviso, pero verificá en una ejecución real que el campo exista antes de depender de él.

## Error workflow (opcional, recomendado para flujos desatendidos)

Un workflow aparte que empieza con `Error Trigger` y notifica. Se asigna en la configuración de cada workflow (`settings.errorWorkflow`). Con el `Stop and Error` puesto, se dispara también cuando la falla vino por una rama de error. Detalle en `n8n-error-handling` → `ERROR_WORKFLOWS.md`.

## Verificación

No lo des por andando sin forzar un error de verdad (por ejemplo, un ID de planilla inválido en una copia de prueba) y comprobar en `n8n_executions`:

1. El aviso llegó y nombra el nodo correcto.
2. La ejecución figura como **error**, no como success.
