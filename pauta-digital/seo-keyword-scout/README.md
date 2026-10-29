# SEO - Keyword Scout

Workflow de n8n que cada 15 días busca **conversaciones reales** (X, Reddit) y **búsquedas reales** (Google Autocomplete) sobre las dos líneas de negocio de Pauta Digital, extrae keywords con intención de compra, las valida contra Google y las deja priorizadas en una planilla, con un resumen por Telegram.

Es el primer eslabón del [pipeline SEO](../seo-pipeline/README.md): el Scout decide **qué** escribir; el Brief Agent, el Writing Agent y el SEO Auditor hacen el resto.

| | |
|---|---|
| Workflow | `SEO - Keyword Scout` (proyecto personal de Pauta Digital en n8n) |
| Estado | Activo desde el 29/10/2026 |
| Disparadores | Días 1 y 15 de cada mes a las 08:17 (ART) · Webhook `POST /webhook/keyword-scout` para correrlo a mano |
| Salida | Planilla `SEO - Banco de keywords (Keyword Scout)` + mensaje de Telegram |
| Líneas de negocio | `consultoria_ia` (consultoría en IA para pymes) · `cursos_automatizacion` (cursos básicos de automatización con IA) |
| Costo medido | ~USD 0,28–0,30 por corrida → ~USD 0,60 por mes |

Los IDs concretos (credenciales, planilla, chat de Telegram) no van en este repo: están en el propio workflow y en la memoria del proyecto.

---

## Flujo

```
Schedule (1 y 15, 08:17) ─┐
Webhook "keyword-scout"  ─┴─▶ Code - Iniciar corrida
  ▶ Sheets - Leer semillas
  ▶ Code - Armar búsquedas            (6 pedidos: por línea, X-es, X-en y Reddit)
  ▶ HTTP - Grok buscar conversaciones (OpenRouter, x-ai/grok-4.3 + plugin web)
  ▶ Code - Unir y verificar hallazgos (filtro anti-invento)
  ▶ Code - Armar autocomplete ▶ HTTP - Google Autocomplete (1 por semilla)
  ▶ Code - Juntar evidencia y armar prompt
  ▶ HTTP - DeepSeek extraer keywords  (OpenRouter, deepseek/deepseek-chat)
  ▶ Sheets - Leer historial ▶ HTTP - Posts de WordPress
  ▶ Code - Validar, dedupe e intención
  ▶ IF - Hay keywords nuevas
       ├─ sí ▶ HTTP - Gemini embeddings ▶ Code - Canibalización semántica
       │      ▶ Code - Elegir top para SERP ▶ HTTP - Serper SERP + PAA
       │      ▶ Code - Armar preguntas ▶ HTTP - Autocomplete preguntas
       │      ▶ Code - Puntaje final ▶ Sheets - Guardar keywords
       │      ▶ Code - Filas de evidencia ▶ Sheets - Guardar evidencia ─┐
       └─ no ──────────────────────────────────────────────────────────┤
  ▶ Code - Fila de corrida ▶ Sheets - Guardar corrida ▶ Telegram - Resumen

Cualquier nodo crítico que falla ▶ Telegram - Aviso de error ▶ Stop and Error
```

### Qué hace cada etapa

1. **Semillas.** Se leen de la pestaña `Semillas` (solo las filas con `activa = si`). Se editan ahí, sin tocar el workflow.
2. **Escucha social (Grok).** Por cada línea de negocio, tres búsquedas: X en español, X en inglés y Reddit (empezando por una lista de subreddits: r/n8n, r/automation, r/ClaudeAI, r/smallbusiness, r/Entrepreneur, r/NegociosArgentina, etc.). Grok **solo junta** posts; no filtra ni clasifica.
3. **Verificación.** Un post queda únicamente si su URL figura entre las fuentes que Grok realmente visitó (`annotations` de la respuesta). Se descartan también los de más de 75 días y los duplicados.
4. **Autocomplete.** Una consulta gratis a Google Autocomplete (Argentina para semillas en español, EE. UU. para las en inglés) por semilla: son búsquedas reales.
5. **Extracción (DeepSeek).** Recibe los posts (`E#`) y las sugerencias (`S#`) y devuelve keywords en español con línea, cluster, tipo de intención, buyer intent 1–5 y los ids de evidencia que las respaldan. Se le exige que al menos la mitad cite posts reales.
6. **Validación en código.** Descarta keywords sin evidencia válida, normaliza (minúsculas, sin acentos, plural básico), deduplica y descarta las que ya están en la planilla.
7. **Canibalización.** Embeddings de Gemini (`gemini-embedding-001`) de cada keyword contra los títulos de los posts publicados en WordPress. Si Gemini falla, cae a coincidencia de palabras (Jaccard).
8. **SERP (Serper).** Para las 30 mejores: top 20 de Google Argentina. Mide qué tan débil es la SERP y si `pautadigital.com.ar` ya aparece en el top 15.
9. **Preguntas.** Autocomplete con prefijos (`como`, `que`, `cuanto`) para las 10 mejores, como reemplazo del People Also Ask (ver Trampas).
10. **Puntaje y guardado.** Puntaje 0–100, filas en `Keywords` y `Evidencia`, registro en `Corridas` y resumen por Telegram.

---

## Reglas de calidad (en código, no en el modelo)

- **Nada sin evidencia.** Un post sin URL verificable se descarta; una keyword sin `E#`/`S#` válido se descarta.
- **Buyer intent** = el máximo entre el puntaje de DeepSeek y el de señales explícitas detectadas por regex:
  - 5: precio, costo, cuánto cuesta/cobra, contratar, presupuesto, quién me lo hace.
  - 4: agencia, consultor/consultoría, curso, implementar, para mi empresa/pyme, alternativas, vs, review, opiniones, confiable, vale la pena, mejor.
  - 3: cómo empezar, desde cero, paso a paso, cómo implementar/automatizar, guía.
  - 2: qué es, cómo funciona, para qué sirve, ejemplos.
- **Puntaje (0–100)** = 45 × intención + 20 × evidencia (hasta 5 piezas) + 25 × debilidad de la SERP + 10 × volumen (neutro, 0,5, hasta tener DataForSEO) + **10 de bonus si el buyer intent es 4 o 5**.
- **Debilidad de la SERP**: promedio sobre el top 10; foros, redes y YouTube = 1; marcas grandes (IBM, HubSpot, Coursera, Globant, Coderhouse, etc.) = 0; el resto = 0,5.
- **Estado**:
  - `refresh` si `pautadigital.com.ar` ya rankea en el top 15 o la similitud con un post propio es ≥ 0,93 → conviene actualizar ese post, no escribir otro.
  - `revisar_canibalizacion` si la similitud es ≥ 0,88.
  - `nueva` en el resto.

---

## Planilla `SEO - Banco de keywords (Keyword Scout)`

Compartida como **Editor** con la cuenta de servicio de n8n (la misma que usa el workflow de Gastos). Si algún nodo de Sheets da 403, lo primero es revisar que siga compartida.

| Pestaña | Para qué | Columnas |
|---|---|---|
| `Semillas` | Temas de partida, editables | activa, linea, semilla, idioma, notas |
| `Keywords` | Banco de keywords (una fila por keyword) | id, fecha, corrida_id, keyword, keyword_norm, idioma, linea, cluster, intencion_tipo, buyer_intent_llm, buyer_intent_reglas, buyer_intent, n_evidencias, fuentes, ejemplo_evidencia, volumen, serp_debilidad, serp_top5, paa, related, mi_posicion, estado, post_similar, similitud, puntaje, usado_en_brief |
| `Evidencia` | Lo que respalda cada keyword | id, fecha, corrida_id, fuente, url, autor, fecha_post, snippet (180 car.), engagement, idioma, verificada, keywords |
| `Corridas` | Log de cada ejecución | corrida_id, fecha, disparador, posts, posts_verificados, candidatas, nuevas, refresh, repetidas_descartadas, costo_usd, avisos |

- `idioma` es el idioma de la **evidencia** (`es`, `en` o `mixto`); la keyword siempre se escribe en español porque el sitio es en español.
- `usado_en_brief` lo completa el Brief Agent con el id del brief cuando usa la keyword (principal o secundaria).
- `engagement` lo reporta Grok y **no se puede verificar**: usarlo como referencia, no como dato.

---

## Cómo usarlo

**Correr a mano** (desde Claude por MCP, o con cualquier cliente HTTP):

```bash
curl -X POST "https://<tu-n8n>/webhook/keyword-scout" -H "Content-Type: application/json" -d '{"linea": "consultoria_ia", "max_serper": 30}'
```

Los dos campos son opcionales:
- `linea`: `consultoria_ia` o `cursos_automatizacion`. Sin él, corre las dos.
- `max_serper`: cuántas keywords van a Serper. Por defecto 30, máximo 60.

**Pasar una keyword al pipeline de redacción:** el Brief Agent toma solo la mejor cada día a las 19:00. Para forzar una: `POST /webhook/brief-agent` con `{"keyword_id": "KS-…"}`. Ver el [README del pipeline](../seo-pipeline/README.md).

**Ajustes frecuentes:**

| Querés cambiar | Dónde |
|---|---|
| Temas | Pestaña `Semillas` |
| Subreddits o modelo de Grok | `Code - Armar búsquedas` (`SUBREDDITS`, `MODELO`) |
| Rúbrica de intención o reglas de extracción | `Code - Juntar evidencia y armar prompt` (`SYSTEM`) |
| Señales explícitas de intención | `Code - Validar, dedupe e intención` (`REGLAS`) |
| Umbrales de canibalización | `Code - Canibalización semántica` (`UMBRAL_EMB`) |
| Pesos del puntaje o dominios fuertes/débiles | `Code - Puntaje final` |
| Frecuencia | `Schedule - Quincenal` |

---

## Costos y rendimiento medidos

| Ítem por corrida | Costo |
|---|---|
| Grok (6 búsquedas vía OpenRouter) | ~USD 0,26–0,28 |
| DeepSeek (1 llamada) | ~USD 0,01–0,02 |
| Google Autocomplete | gratis |
| Gemini embeddings | plan gratis |
| Serper (30 búsquedas) | créditos gratis (1 crédito por búsqueda con `num: 20`) |
| **Total** | **~USD 0,28–0,30** |

### Pruebas del 29/10/2026

| Corrida | Posts (crudos → verificados) | Keywords nuevas | Respaldadas por X/Reddit | Problemas encontrados |
|---|---|---|---|---|
| KS-20261029-1138 | 13 → 13 | 10 | 1 de 10 | Telegram falló; 3 de 6 búsquedas de Grok vacías; umbrales de canibalización mal calibrados; sin PAA |
| KS-20261029-1208 | 38 → 36 | 35 | 0 de 35 | DeepSeek ignoró los posts; Serper rechazó 5 de 30 por rate limit |
| KS-20261029-1451 | 45 → 26 | 15 (+9 repetidas descartadas) | **8 de 15** | Sin fallas; X en español para cursos sigue sin resultados |

Las filas de la primera corrida se borraron de la planilla (tenían mal calculado el estado de canibalización). Las de la segunda y la tercera quedaron.

---

## Trampas encontradas

Todas pasaron en las pruebas de este workflow.

| Síntoma | Causa | Cómo quedó resuelto |
|---|---|---|
| Serper devuelve `400 Query pattern not allowed for free accounts` | El plan gratis de Serper no acepta el operador `site:` | Reddit se busca con Grok limitado a `reddit.com` |
| La API JSON de Reddit devuelve una página HTML de bloqueo | Reddit bloquea las IP del VPS | Igual que arriba. Alternativa a futuro: un actor de Apify |
| Serper no trae `peopleAlsoAsk` ni `relatedSearches` | Para Google Argentina devuelve solo resultados orgánicos (confirmado en 26 búsquedas) | Preguntas reales de Google Autocomplete con prefijos |
| Serper responde *"Try spacing your requests"* | Rate limit con 5 pedidos cada 0,5 s | Lotes de 2 cada 1,5 s |
| Grok responde `{"posts": []}` aunque visitó 24–37 fuentes | El prompt pedía "solo preguntas"; Grok prefiere vacío a relajar el filtro | El prompt pide incluir todo post relevante; el filtrado lo hace DeepSeek |
| DeepSeek arma todas las keywords con Autocomplete e ignora los posts | Copiar sugerencias es el camino fácil | El prompt exige que al menos la mitad cite `E#` y limita a 3 variantes informacionales por cluster |
| Todo sale como "posible canibalización" | Con `gemini-embedding-001`, dos textos del mismo rubro sin relación ya dan 0,79–0,84 de similitud | Umbrales en 0,88 (revisar) y 0,93 (refresh) |
| Telegram: *"can't parse entities"* | El texto tenía guiones bajos (`consultoria_ia`) que Telegram interpretó como formato | `parse_mode: HTML` y escape de `<`, `>` y `&` en el texto |
| El nodo de Telegram de resumen falla y la ejecución queda en rojo sin aviso | Era el último nodo y no tenía salida de error | Salida de error cableada al aviso |

---

## Pendientes

- [ ] **Volumen de búsqueda** con DataForSEO (etapa 1.5). Hoy la columna `volumen` está vacía y pesa neutro en el puntaje.
- [ ] **X en español para cursos**: no trajo posts en ninguna de las tres corridas. Si sigue así, cambiar esa búsqueda por otra semilla o por otro idioma.
- [ ] **Preguntas por keyword**: Autocomplete con prefijos casi no devuelve nada para keywords largas. Evaluar otra fuente de PAA real.
- [ ] **Calibrar umbrales** de canibalización con más corridas.
- [x] **Etapa 3**: brief con entidades, fan-out de IA y notas GEO, conectado al Scout. Quedó como el [pipeline SEO](../seo-pipeline/README.md) (Brief Agent → Writing Agent → SEO Auditor).
- [ ] Opcional: asignar un error workflow en la configuración del workflow (no se puede por MCP).

Primer brief generado con estos datos: [`brief-01-agencia-automatizacion-ia-pymes.md`](brief-01-agencia-automatizacion-ia-pymes.md).
