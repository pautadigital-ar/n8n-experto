# Pipeline SEO de Pauta Digital: Brief Agent → Writing Agent → SEO Auditor

Tres workflows de n8n que convierten el banco de keywords del [Keyword Scout](../seo-keyword-scout/README.md) en artículos programados en WordPress, con una sola intervención humana: revisar el borrador y pasarlo a **Pendiente**.

```
Keyword Scout (1 y 15 de cada mes)
   │  banco de keywords en Sheets
   ▼
Brief Agent (todos los días, 19:00)      → fila en la pestaña Briefs (estado brief_listo)
   ▼
Writing Agent (todos los días, 20:00)    → borrador "[VER] …" en WordPress (estado borrador)
   ▼
Sebas revisa el borrador y lo pasa a Pendiente
   ▼
SEO Auditor (cada hora, 8 a 23 hs)       → aprobado: programado para el sábado 15:00
                                          → bloqueado: vuelve a Borrador con los motivos
```

| Workflow | Disparadores | Modelo | Costo medido |
|---|---|---|---|
| `SEO - Brief Agent` | Diario 19:00 (ART) · `POST /webhook/brief-agent` | DeepSeek V4.1 Flash (OpenRouter) | ~USD 0,01 por brief + 1 crédito de Serper |
| `SEO - Writing Agent` | Diario 20:00 (ART) · `POST /webhook/writing-agent` | DeepSeek V4.1 Flash (OpenRouter) | ~USD 0,02–0,04 por borrador |
| `SEO - Auditor` | Minuto 7 de cada hora, 8 a 23 (ART) · `POST /webhook/seo-auditor` | Ninguno (reglas en código) | Gratis |

Con un artículo por día: **~USD 1,50 por mes** entre brief y redacción, más ~USD 0,60 del Scout.

Todos están activos desde el 29/10/2026 en el proyecto personal de Pauta Digital en n8n. Los IDs concretos (credenciales, planilla, chat de Telegram) no van en este repo: están en los propios workflows.

---

## Lo que hace Sebas (la única parte manual)

1. Llega por Telegram el aviso **"Borrador listo"** con el link de edición, las marcas pendientes y las afirmaciones a verificar.
2. En WordPress, el post aparece como **`[VER] Título…`** en estado Borrador.
3. Revisarlo:
   - Completar cada `[CASO: …]` con un caso real, o borrar la frase.
   - Chequear la lista **verificar** (columna `verificar` de la pestaña Briefs y mensaje de Telegram).
   - Leerlo: el Auditor **no** evalúa la calidad del texto (ver más abajo).
4. Elegir **Pendiente** en el estado del post y tocar **Guardar**. Sin Guardar, sigue en Borrador y el Auditor no lo ve.
5. El Auditor lo agarra en la próxima pasada (o al momento, corriendo su webhook) y avisa por Telegram si lo programó o lo devolvió a Borrador.

El `[VER]` del título se puede borrar a mano o dejar: el Auditor lo saca al programar.

---

## 1. SEO - Brief Agent

Elige la keyword del día, investiga la SERP real y arma un brief completo en JSON.

### Flujo

```
Schedule (19:00) ─┐
Webhook          ─┴─▶ Code - Iniciar brief
  ▶ Sheets - Leer briefs ▶ Sheets - Leer banco ▶ Code - Elegir keyword
  ▶ IF - Pausado
       ├─ sí ▶ Telegram - Aviso de pausa
       └─ no ▶ Sheets - Leer evidencia ▶ Code - Evidencia de la keyword
              ▶ HTTP - Serper top 10 ▶ Code - Elegir páginas ▶ HTTP - Leer páginas del top
              ▶ Code - Analizar competidores ▶ Code - Armar preguntas ▶ HTTP - Autocomplete
              ▶ HTTP - Posts de WordPress ▶ Code - Armar dossier ▶ HTTP - DeepSeek brief
              ▶ Code - Validar brief ▶ Code - Fila del brief ▶ Sheets - Guardar brief
              ▶ Code - Keywords a marcar ▶ Sheets - Marcar keywords usadas
              ▶ Code - Mensaje del brief ▶ Telegram - Brief listo

Cualquier nodo crítico que falla ▶ Telegram - Aviso de error ▶ Stop and Error
```

### Qué hace

1. **Freno.** Si hay 3 briefs en estado `bloqueado_auditoria`, no genera más y avisa: primero hay que resolver los trabados.
2. **Keyword.** La de mayor puntaje del banco con estado `nueva`, sin usar y buyer intent ≥ 3. Suma hasta 5 secundarias del mismo cluster y línea. Si no queda ninguna, avisa y espera la próxima corrida del Scout.
3. **Evidencia.** Los posts de X/Reddit y las sugerencias de Autocomplete que respaldan esas keywords, con refs citables `V#`.
4. **SERP.** Top 10 de Google Argentina (Serper) y lectura del HTML de hasta 8 competidores: título, H1/H2/H3, palabras, FAQ, tablas, schema y fecha.
5. **Preguntas.** Autocomplete con prefijos (`como`, `que`, `cuanto`, `por que`, `vale la pena`) sobre la principal y 2 secundarias.
6. **Enlaces internos.** Solo posts publicados reales de WordPress.
7. **Brief.** DeepSeek arma el JSON: título, slug, meta description, ángulo, hueco detectado, secciones con H2/H3, talking points y largo, preguntas, entidades, notas GEO, enlaces internos, CTA y largo total.
8. **Validación en código.** Campos obligatorios, reglas de Rank Math (keyword en los primeros 50 caracteres del título, título ≤ 65, meta 120–155 con la keyword, keyword en al menos 2 H2), refs y enlaces que existan. Las cifras que no aparecen en la investigación no se borran: van a la columna `verificar`.
9. **Guardado.** Fila en `Briefs` con estado `brief_listo`; la principal y las secundarias quedan con `usado_en_brief` = id del brief.

**Correr a mano:** `POST /webhook/brief-agent` con `{"keyword_id": "KS-…"}` o `{"keyword": "…"}`. Sin body elige solo.

---

## 2. SEO - Writing Agent

Redacta el artículo por secciones, lo pasa por un editor y lo sube como **borrador** a WordPress. Nunca publica.

### Flujo

```
Schedule (20:00) ─┐
Webhook          ─┴─▶ Code - Iniciar redacción
  ▶ Sheets - Leer briefs ▶ Code - Elegir brief ▶ IF - Hay brief
  ▶ Sheets - Leer casos ▶ Code - Preparar secciones
  ▶ HTTP - Redactar sección (de a 3 en paralelo)
  ▶ Code - Ensamblar artículo ▶ HTTP - Editar artículo ▶ Code - Normalizar edición
  ▶ HTTP - WordPress crear borrador ▶ Code - Fila del borrador ▶ Sheets - Actualizar brief
  ▶ HTTP - Rank Math meta ▶ Code - Mensaje del borrador ▶ Telegram - Borrador listo

Cualquier nodo crítico que falla ▶ Telegram - Aviso de error ▶ Stop and Error
```

### Qué hace

1. **Brief.** El pedido a mano o el `brief_listo` más viejo. Si un día falla, al siguiente se pone al día.
2. **Casos reales.** Lee la pestaña `Casos` (ver abajo).
3. **Redacción por partes.** Un pedido al modelo por intro, cada H2, preguntas frecuentes y conclusión. Cada parte conoce el esquema completo para no repetir lo que le toca a otra. El largo total del brief se reparte proporcionalmente entre las secciones.
4. **Preguntas frecuentes.** Una sola sección: si el brief ya trae una, las preguntas se responden ahí y no se agrega otra al final.
5. **Ensamblado.** Une las partes en orden, pone `id` a los H2 y agrega la tabla de contenidos. Si una sección vino vacía, corta con aviso.
6. **Editor.** Una pasada sobre el texto completo: repeticiones entre secciones, legibilidad, voseo, una misma cifra como máximo dos veces, clichés prohibidos. Devuelve también la lista de afirmaciones a verificar. Si el editor falla o recorta más del 10%, queda la versión sin editar.
7. **WordPress.** Borrador con título `[VER] …`, slug, extracto y contenido. Después, la keyword y la meta description de Rank Math por su propio endpoint.
8. **Planilla y aviso.** La fila del brief pasa a `borrador` con `wp_post_id`, `borrador_url`, `palabras`, `marcadores` y `verificar`. Telegram con el resumen.

### Reglas anti-invento (en el prompt del redactor)

- Ninguna cifra, porcentaje, estudio ni cliente que no esté en el brief o la evidencia.
- Los datos de `verificar` se escriben atribuidos a su fuente ("según…"), nunca como afirmación propia.
- Casos propios: solo los de la pestaña `Casos`. Si ninguno encaja, el marcador literal `[CASO: qué caso iría]`.
- Enlaces: solo URLs del brief. El CTA final va a `/servicios/` (consultoría y cursos).

**Correr a mano:** `POST /webhook/writing-agent` con `{"brief_id": "BR-…"}` (opcional).

---

## 3. SEO - Auditor

El último eslabón. Revisa con **reglas fijas en código** los posts que Sebas pasó a Pendiente y decide si se programan.

### Flujo

```
Schedule (:07, 8 a 23 hs) ─┐
Webhook                   ─┴─▶ HTTP - Posts pendientes ▶ Sheets - Leer briefs
  ▶ Code - Revisar posts ▶ Code - Enlaces a chequear ▶ HTTP - Chequear enlace
  ▶ Code - Decidir ▶ HTTP - WordPress actualizar post ▶ Code - Filas de auditoría
  ▶ Sheets - Actualizar briefs ▶ Code - Mensaje de auditoría ▶ Telegram - Resultado

Cualquier nodo crítico que falla ▶ Telegram - Aviso de error ▶ Stop and Error
```

Solo audita posts Pendientes que tienen fila en la pestaña `Briefs`. Un post Pendiente que no salió del pipeline se ignora: el Auditor nunca programa algo que no armó el Writing Agent.

### Controles

| Bloquea (vuelve a Borrador) | Avisa (no frena) |
|---|---|
| Marcas `[CASO]`, `[LINK]` o `[VER]` sin completar | Keyword fuera de los primeros 50 caracteres del título |
| Restos de código (```` ``` ````, `{{ }}`, `<html>`, `<body>`) | Título de más de 65 caracteres |
| Menos de 800 palabras | Slug distinto del brief |
| Slug terminado en número (`-2`, `-3`…) | Keyword ausente al principio del artículo |
| Más de una sección de preguntas frecuentes | Una misma cifra más de 2 veces en el texto (sin contar tablas, tabla de contenidos ni años) |
| Títulos de sección repetidos | Enlaces que responden 403, 429 o 5xx |
| Enlaces 404/410 o dominios que no responden | |
| Post sin título | |

**Aprobado:** saca el `[VER]` del título y lo programa (`status: future`) para el próximo sábado a las 15:00, con 15 minutos entre posts del mismo sábado. En la planilla: estado `programado`, `publicar_el` y "Aprobado sin observaciones" (o los avisos) en `auditoria`.

**Bloqueado:** vuelve a Borrador; estado `bloqueado_auditoria` y los motivos en `auditoria`. Con 3 bloqueados, el Brief Agent se pausa.

### Lo que el Auditor NO hace

No lee el artículo como un editor: no detecta relleno, ejemplos con cifras inventadas presentadas como hipotéticas, afirmaciones sin respaldo ni texto flojo. Una pasada completa tarda unos 5 segundos porque son solo reglas. Una revisión con IA (marcar relleno, cifras sin fuente y afirmaciones flojas, ~centavos por artículo) quedó propuesta y **no** está implementada.

**Correr a mano:** `POST /webhook/seo-auditor` (sin body).

---

## Planilla

La misma del Scout (`SEO - Banco de keywords (Keyword Scout)`), compartida como **Editor** con la cuenta de servicio de n8n. Pestañas que usa el pipeline:

| Pestaña | Quién escribe | Columnas |
|---|---|---|
| `Keywords` | Scout; el Brief Agent completa `usado_en_brief` | ver el README del Scout |
| `Evidencia` | Scout | ver el README del Scout |
| `Briefs` | Brief Agent crea la fila; Writing Agent y Auditor la actualizan por `id` | id, fecha, keyword_id, keyword, linea, cluster, keywords_secundarias, tipo_contenido, titulo, slug, estado, brief_json, investigacion_json, verificar, costo_usd, avisos, wp_post_id, publicar_el, borrador_url, palabras, marcadores, auditoria |
| `Casos` | Sebas, a mano | activo, rubro, situacion_inicial, que_hicimos, resultado, linea, se_puede_nombrar, nombre_o_ciudad |

**Estados de un brief:** `brief_listo` → `borrador` → `programado` o `bloqueado_auditoria`.

### Pestaña `Casos`

Casos reales de clientes para que el redactor los use como ejemplo propio en lugar de dejar `[CASO: …]`.

- Solo cuentan las filas con `activo` = `si`. La fila 2 de ejemplo (`activo` = `ejemplo`) se ignora.
- El redactor usa primero los de la misma `linea` que el artículo (`consultoria_ia` o `cursos_automatizacion`), hasta 8.
- Cuenta el caso con los datos de la fila, sin agregar cifras, y nombra al cliente solo si `se_puede_nombrar` = `si`.
- Si la pestaña está vacía o la lectura falla, el artículo se escribe igual con marcadores `[CASO: …]`.

---

## Ajustes frecuentes

| Querés cambiar | Dónde |
|---|---|
| Hora del brief / del borrador | `Schedule - Diario 19 hs` / `Schedule - Diario 20 hs` |
| Intención mínima o tamaño del freno | Brief Agent → `Code - Elegir keyword` (`INTENCION_MINIMA`, `MAX_TRABADOS`) |
| Prompt del brief | Brief Agent → `Code - Armar dossier` |
| Reglas de Rank Math del brief | Brief Agent → `Code - Validar brief` |
| Voz, reglas anti-invento o modelo del redactor | Writing Agent → `Code - Preparar secciones` (`SYSTEM`, `MODELO`) |
| Destino del CTA | Writing Agent → `Code - Preparar secciones` (`CTA_URL`) |
| Reglas del editor | Writing Agent → `Code - Ensamblar artículo` (`EDITOR`) |
| Controles del Auditor | Auditor → `Code - Revisar posts` (bloqueos y avisos) y `Code - Decidir` (enlaces) |
| Día y hora de publicación | Auditor → `Code - Decidir` (próximo sábado 15:00, +15 min por post) |

---

## Primera vuelta completa (29/10/2026)

| Paso | Resultado |
|---|---|
| Brief `BR-20261029-1601` | Keyword "costo de agencia de ia" (+ secundarias del cluster). USD 0,0065 |
| Borrador 277 | Primera prueba: 4196 palabras contra 2200 pedidas y el editor devolvió vacío. Descartado (papelera) |
| Borrador 278 | Con las correcciones: Rank Math guardado, USD 0,0378. Salió con slug `-2` porque el 277 ocupaba el slug |
| Revisión manual | Sebas completó los `[CASO]`, corrigió el slug y lo pasó a Pendiente |
| Auditor | Aprobado sin observaciones: 2974 palabras, 4 enlaces con 200 OK. Programado para el sábado 31/10 a las 15:00 |

Leyendo el 278 después de aprobado aparecieron problemas que las reglas no cubrían: dos secciones de preguntas frecuentes seguidas, el mismo dato (87,3%) repetido 5 veces, un ejemplo con montos inventados y una afirmación sin respaldo. Las dos secciones de preguntas y los datos repetidos se corrigieron en el Writing Agent y quedaron como controles del Auditor; el resto es lo que cubriría la revisión con IA.

---

## Trampas encontradas

| Síntoma | Causa | Cómo quedó resuelto |
|---|---|---|
| El brief llega con el JSON cortado | DeepSeek V4.1 Flash gasta `max_tokens` razonando antes de responder | `max_tokens: 16000` y `reasoning: { effort: 'low' }` |
| El editor devuelve el texto vacío | Mismo motivo: usó los 16000 tokens pensando | Editor con `reasoning: { enabled: false }` |
| El validador marca "2.227" como cifra inventada | Separador de miles distinto entre el brief y la investigación | Normalización de números antes de comparar |
| Artículo de 4196 palabras para 2200 pedidas | Cada sección se escribía con su largo del brief más intro, FAQ y conclusión | Largo repartido proporcionalmente (intro 250, FAQ 300, conclusión 180, el resto escalado). Todavía sale ~35% largo |
| La keyword y la meta de Rank Math no se guardan | WordPress ignora `rank_math_*` en `/wp/v2/posts`: no están registrados para REST | Nodo aparte a `POST /wp-json/rankmath/v1/updateMeta`. Respuesta OK: `{"slug":true,"schemas":[]}` |
| El slug sale con `-2` | Otro post, aunque esté en la papelera, ya usa ese slug | El Auditor lo bloquea; hay que vaciar la papelera y corregir el slug |
| WordPress no tiene estado "revisado" | Los estados nativos son Borrador, Pendiente, Programada y Publicada | La señal de revisión es **Pendiente** + el prefijo `[VER]` en el título |
| El Auditor no encuentra el post Pendiente | Se eligió Pendiente en el menú pero no se tocó Guardar | Guardar después de cambiar el estado |
| Dos secciones de preguntas frecuentes | El brief traía su sección de FAQ y el redactor agregaba otra | Una sola sección (Writing Agent) y bloqueo en el Auditor |
| Un nodo agregado por API aparece arriba a la izquierda | Sin `position`, n8n lo pone en `[0,0]` | Pasar `position` al crearlo o moverlo con `setNodePosition` |
| Un webhook creado por API no responde | Le falta `webhookId` | `webhookId` explícito en cada nodo Webhook |
| No se puede usar un Telegram Trigger con Gordo_bot | Un bot admite un solo webhook y lo tiene el pipeline SEO viejo | Los workflows nuevos solo envían mensajes; se disparan por Schedule o Webhook |

---

## Pendientes

- [ ] **Revisión con IA en el Auditor** (propuesta, sin implementar): relleno, repeticiones, cifras sin fuente y afirmaciones flojas, con sugerencias por Telegram.
- [ ] **Largo**: los artículos salen ~35% más largos que lo pedido.
- [ ] **Casos**: cargar casos reales en la pestaña `Casos` (`activo` = `si`).
- [ ] **CTA de cursos**: hoy va a `/servicios/`; cambiar `CTA_URL` si se arma una página propia.
- [ ] **Keywords `refresh`**: el Scout las marca pero ningún workflow las trabaja todavía (actualizar el post existente en lugar de escribir otro).
- [ ] **Monitoreo post-publicación** con Search Console.
- [ ] **Volumen** con DataForSEO (etapa 1.5 del Scout).
- [ ] Opcional: asignar un error workflow en la configuración de cada workflow (no se puede por MCP).
