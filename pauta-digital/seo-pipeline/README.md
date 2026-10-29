# Pipeline SEO de Pauta Digital: Brief Agent → Writing Agent → Internal Linking → Portada → SEO Auditor → Indexación

Seis workflows de n8n que convierten el banco de keywords del [Keyword Scout](../seo-keyword-scout/README.md) en artículos programados en WordPress, con una sola intervención humana: revisar el borrador y pasarlo a **Pendiente**.

```
Keyword Scout (1 y 15 de cada mes)
   │  banco de keywords en Sheets
   ▼
Brief Agent (todos los días, 19:00)      → fila en la pestaña Briefs (estado brief_listo)
   ▼
Writing Agent (todos los días, 20:00)    → borrador "[VER] …" en WordPress (estado borrador)
   ▼
Internal Linking, modo borrador          → 3 a 6 enlaces del borrador hacia posts publicados
   ▼
Portada                                  → imagen destacada con la plantilla de marca + aviso "Borrador listo" con la foto
   ▼
Sebas revisa el borrador y lo pasa a Pendiente
   ▼
SEO Auditor (cada hora, 8 a 23 hs)       → aprobado: programado para el sábado 15:00
                                          → bloqueado: vuelve a Borrador con los motivos
   ▼
Internal Linking, modo publicado         → cuando sale el post, enlace desde hasta 3 posts viejos (estado publicado)
   ▼
Indexación                               → reenvía el sitemap a Search Console; revisión diaria; sábados: lista para pedir a mano
```

| Workflow | Disparadores | Modelo | Costo medido |
|---|---|---|---|
| `SEO - Brief Agent` | Diario 19:00 (ART) · `POST /webhook/brief-agent` | DeepSeek V4.1 Flash (OpenRouter) | ~USD 0,01 por brief + 1 crédito de Serper |
| `SEO - Writing Agent` | Diario 20:00 (ART) · `POST /webhook/writing-agent` | DeepSeek V4.1 Flash (OpenRouter) | ~USD 0,02–0,04 por borrador |
| `SEO - Internal Linking` | Lo llama el Writing Agent · minuto 37 de cada hora, 8 a 23 (ART) · `POST /webhook/internal-linking` | DeepSeek V4.1 Flash + embeddings de Gemini | ~USD 0,002–0,01 por pasada |
| `SEO - Portada` | Lo llama Internal Linking · `POST /webhook/portada` | DeepSeek V4.1 Flash (solo textos) + Gotenberg | ~USD 0,001 por portada |
| `SEO - Indexación` | Lo llama Internal Linking al publicarse · diario 09:23 (ART) · `POST /webhook/indexacion` | Ninguno (API de Search Console) | Gratis |
| `SEO - Auditor` | Minuto 7 de cada hora, 8 a 23 (ART) · `POST /webhook/seo-auditor` | Ninguno (reglas en código) | Gratis |

Con un artículo por día: **~USD 1,60 por mes** entre brief, redacción, enlaces y portada, más ~USD 0,60 del Scout.

Todos están activos desde el 29/10/2026 en el proyecto personal de Pauta Digital en n8n. Los IDs concretos (credenciales, planilla, chat de Telegram) no van en este repo: están en los propios workflows.

---

## Lo que hace Sebas (la única parte manual)

1. Llegan por Telegram la **portada** y el aviso **"Borrador listo"** con el link de edición, las marcas pendientes, las afirmaciones a verificar y los enlaces internos que se agregaron.
2. En WordPress, el post aparece como **`[VER] Título…`** en estado Borrador.
3. Revisarlo:
   - Completar cada `[CASO: …]` con un caso real, o borrar la frase.
   - Chequear la lista **verificar** (columna `verificar` de la pestaña Briefs y mensaje de Telegram).
   - Leerlo: el Auditor **no** evalúa la calidad del texto (ver más abajo).
4. Elegir **Pendiente** en el estado del post y tocar **Guardar**. Sin Guardar, sigue en Borrador y el Auditor no lo ve.
5. El Auditor lo agarra en la próxima pasada (o al momento, corriendo su webhook) y avisa por Telegram si lo programó o lo devolvió a Borrador.

El `[VER]` del título se puede borrar a mano o dejar: el Auditor lo saca al programar.

**Los sábados a la mañana** llega por Telegram la lista de posts que Google todavía no indexó después de 3 días: en cada link de la lista, **"Solicitar indexación"** en Search Console, y después `si` en la columna `pedido` de la pestaña `Indexación`.

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

## 3. SEO - Internal Linking

Agrega enlaces internos en dos momentos. El modelo solo propone; las reglas las pone el código.

### Flujo

```
Webhook (Writing Agent) ─┐
Schedule (:37, 8 a 23)  ─┴─▶ Code - Iniciar ▶ Sheets - Leer briefs ▶ Code - Elegir post ▶ IF - Hay post
  ▶ HTTP - Post nuevo ▶ HTTP - Posts publicados ▶ Code - Preparar embeddings ▶ HTTP - Gemini embeddings
  ▶ Code - Armar pedido ▶ HTTP - DeepSeek enlaces ▶ Code - Validar y aplicar ▶ IF - Hay cambios
       ├─ sí ▶ HTTP - WordPress guardar ─┐
       └─ no ────────────────────────────┴▶ Code - Resumen ▶ Sheets - Actualizar brief ▶ IF - Modo borrador
                                                   ├─ borrador ▶ HTTP - Pasar a portada (si falla ▶ Telegram - Borrador listo sin portada)
                                                   └─ publicado ▶ Telegram - Resultado

Cualquier nodo crítico que falla ▶ Telegram - Aviso de error ▶ Stop and Error
```

### Modo borrador (todas las noches, después del Writing Agent)

1. El Writing Agent llama al webhook con el id del brief y el texto del aviso.
2. Busca los posts publicados más parecidos al borrador (embeddings de Gemini; si fallan, palabras en común).
3. DeepSeek propone, para hasta 8 destinos, una frase que ya está escrita en el borrador.
4. El código aplica entre 3 y 6 y le pasa la posta a **SEO - Portada** con el texto del aviso más la lista de enlaces. Si este agente falla, el aviso sale igual (desde el Writing Agent o desde el aviso de error) con el borrador sin enlaces nuevos; si falla la llamada a Portada, el aviso sale desde acá sin portada.

### Modo publicado (cada hora)

1. Busca briefs `programado` cuya hora de publicación ya pasó. Si WordPress todavía no publicó el post, espera; con más de 90 minutos de atraso avisa (el cron de WordPress se atrasa en sitios con poco tráfico).
2. Elige hasta 6 posts viejos relacionados, sin los armados con maquetador (Elementor, Divi, WPBakery, Avada) ni los que ya enlazan al nuevo.
3. DeepSeek propone una frase por post viejo; el código aplica **como máximo 3, un enlace por post**.
4. El brief pasa a `publicado` con `publicado_el` y `enlaces_entrantes`, y llega un aviso con los posts cambiados y el link para editarlos.

### Reglas en código

- La frase existe tal cual (sin distinguir mayúsculas) y **una sola vez**, en palabras completas.
- Fuera de títulos, de otros enlaces, de código y de comentarios de bloques de WordPress.
- De 2 a 8 palabras; ni anclas genéricas ("acá", "este artículo", "leer más"…) ni listas separadas por comas.
- Un enlace por destino; no repite destinos que el post ya enlaza.
- El texto visible del post tiene que quedar **idéntico**: solo se envuelve la frase en `<a>`.
- Para deshacer un cambio: en el editor del post → **Revisiones**.

**Correr a mano:** `POST /webhook/internal-linking` con `{"modo": "borrador", "brief_id": "BR-…"}` o `{"modo": "publicado", "brief_id": "BR-…"}`.

---

## 4. SEO - Portada

Genera la imagen destacada del borrador con la **plantilla de marca aprobada**: no hay modelo de imagen; la IA solo escribe los textos y elige el sticker.

### Flujo

```
Webhook (Internal Linking) ▶ Code - Iniciar ▶ Sheets - Leer briefs ▶ Sheets - Leer stickers ▶ Google Drive - Listar stickers
  ▶ Code - Armar pedido ▶ HTTP - DeepSeek portada ▶ Code - Validar portada ▶ Google Drive - Bajar sticker
  ▶ Code - Componer HTML ▶ HTTP - Gotenberg render ▶ HTTP - WordPress subir imagen ▶ HTTP - WordPress texto alternativo
  ▶ HTTP - WordPress poner destacada ▶ Code - Resumen ▶ Sheets - Actualizar brief ▶ Telegram - Foto de la portada ▶ Telegram - Borrador listo

Cualquier nodo crítico que falla ▶ Telegram - Aviso de error (incluye el aviso del borrador) ▶ Stop and Error
```

### Qué hace

1. **Stickers disponibles:** los que tienen fila con `activo` = `si` en la pestaña `Stickers` **y** archivo en la carpeta de Drive "Stickers portadas (agente)" (compartida con la cuenta de servicio). Si hay archivos sin fila o filas sin archivo, lo avisa.
2. **Textos (DeepSeek, sin razonamiento):** 3 opciones de titular corto en dos líneas (`linea1` en hueso, `resaltado` en cyan), rótulo coral (`GUÍA PRÁCTICA`, `PRECIOS 2026`…), sticker según los `temas` y texto alternativo.
3. **Reglas en código:**
   - **Tamaño del titular según la línea más larga:** hasta 14 caracteres → 150 px; 15 a 17 → 125 px; 18 a 20 → 110 px. Más de 20 no entra: se usa otra de las 3 opciones o se reparten distinto las mismas palabras; si nada baja de 21, la portada no se genera y llega el aviso. El line-height (0,95), el letter-spacing (−0,035em), la posición (x 96) y los 40 px hasta el rótulo no cambian.
   - Sticker de la lista (si el modelo inventa uno, el de más temas en común), rótulo en mayúsculas, sin signos de exclamación ni emojis.
4. **HTML con la plantilla:** 1600×900, fondo #0b1a1d con grilla y halos cyan (capas explícitas: fondo → grilla → halos → textos → sticker), titular Space Grotesk 700, rótulo coral rotado −3°, sticker a la derecha rotado 6°, "01 // BLOG" (700) y "pauta digital" (500). **Una sola tipografía para todo**, sin ninguna otra `font-family`: Space Grotesk se baja de Google Fonts en cada corrida y se incrusta.
5. **En el navegador, antes de la captura:**
   - Recorta los márgenes transparentes del sticker (cuenta solo filas y columnas con 8+ píxeles opacos, así ignora restos del recorte de fondo).
   - Agrega la sombra dura cyan **solo si el sticker no la trae dibujada** (mira si el borde derecho ya es cyan).
   - Carga y verifica la fuente en cada peso y tamaño que usa la plantilla (`document.fonts.load` + `document.fonts.check`). Si alguno falla, no marca la página como lista: Gotenberg corta por tiempo y llega el aviso, en vez de capturar con otra tipografía.
6. **Gotenberg** (Easypanel, `http://n8n_gotenberg:3000`, el mismo de los carruseles) la convierte en WebP esperando `data-listo`.
7. **WordPress:** sube la imagen (`{slug}-portada.webp`), le pone alt y título, y la deja como **imagen destacada** del borrador.
8. **Aviso:** la foto por Telegram y el mensaje "Borrador listo" con el titular, el rótulo y el sticker usados.

### Stickers

| Dónde | Qué |
|---|---|
| Drive: "Portadas Blog / Stickers portadas (agente)" | Los archivos. Mejor SVG; sirve PNG con fondo transparente de 1200×1200 o más |
| Planilla, pestaña `Stickers` | `activo` (`si`), `archivo` (nombre exacto), `que_es`, `temas` (de qué artículos puede ser portada: es lo que usa la IA para elegir) |

Un sticker nuevo está disponible en la corrida siguiente a subirlo y cargar su fila.

**Correr a mano:** `POST /webhook/portada` con `{"brief_id": "BR-…"}`. Para regenerar con textos fijos (sin IA): `{"brief_id", "linea1", "resaltado", "rotulo", "sticker": "archivo.png"}`.

---

## 5. SEO - Auditor

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
| Post sin imagen destacada (portada) | |

**Aprobado:** saca el `[VER]` del título y lo programa (`status: future`) para el próximo sábado a las 15:00, con 15 minutos entre posts del mismo sábado. En la planilla: estado `programado`, `publicar_el` y "Aprobado sin observaciones" (o los avisos) en `auditoria`.

**Bloqueado:** vuelve a Borrador; estado `bloqueado_auditoria` y los motivos en `auditoria`. Con 3 bloqueados, el Brief Agent se pausa.

### Lo que el Auditor NO hace

No lee el artículo como un editor: no detecta relleno, ejemplos con cifras inventadas presentadas como hipotéticas, afirmaciones sin respaldo ni texto flojo. Una pasada completa tarda unos 5 segundos porque son solo reglas. Una revisión con IA (marcar relleno, cifras sin fuente y afirmaciones flojas, ~centavos por artículo) quedó propuesta y **no** está implementada.

**Correr a mano:** `POST /webhook/seo-auditor` (sin body).

---

## 6. SEO - Indexación

Sigue cada post publicado hasta que Google lo indexa. **Sin Indexing API**: Google la permite solo para ofertas de empleo y transmisiones en vivo, y usarla para un blog arriesga perder el acceso. Bing (y ChatGPT, que busca con Bing) lo cubre **IndexNow de Rank Math**, que avisa solo al publicar o actualizar.

### Flujo

```
Webhook (Internal Linking, modo publicado) ─┐
Schedule (09:23)                            ─┴─▶ Code - Iniciar ▶ HTTP - Propiedades de Search Console ▶ Sheets - Leer indexación
  ▶ Code - Planificar ▶ IF - Modo publicado
       ├─ publicado ▶ HTTP - Reenviar sitemap ▶ Code - Fila nueva ▶ Sheets - Guardar fila nueva ▶ IF - Falló el sitemap ▶ Telegram
       └─ revisar   ▶ HTTP - Inspeccionar URL ▶ Code - Resultados ▶ Sheets - Actualizar indexación ▶ IF - Hay algo para avisar ▶ Telegram - Resumen

Cualquier nodo crítico que falla ▶ Telegram - Aviso de error ▶ Stop and Error
```

- **Propiedad:** se elige sola entre las que ve la cuenta de servicio con permiso Completo (hoy `sc-domain:pautadigital.com.ar`).
- **Al publicarse:** reenvía `https://pautadigital.com.ar/sitemap_index.xml` (`sitemaps.submit`, responde 204) y abre la fila con `accion` = `esperar`.
- **Todos los días 09:23:** inspecciona con la URL Inspection API cada URL no indexada con 2 días o más (cupo de 2000 por día; se usan unas pocas). Anota el estado que da Google (en español), el último rastreo y la acción.
- **Acciones:** `ok` (indexado) · `esperar` (menos de 3 días) · `pedir a mano` (3 días o más) · `pedido` (Sebas puso `si` en `pedido`: el agente le pone fecha y no lo vuelve a listar por 7 días) · `revisar` (14 días o más sin indexar: contenido flojo o canibalización).
- **Avisos:** cuando algo se indexa; los sábados, la lista para pedir a mano con el link directo a la inspección de cada URL; y una vez cuando algo pasa a `revisar`. Sin novedades, no manda nada.

**Configuración en Google (hecha el 29/10/2026):** Search Console API habilitada en el proyecto de la cuenta de servicio; la cuenta de servicio como usuario con permiso **Completo** en la propiedad (reenviar sitemaps no funciona con permiso restringido); en n8n, la credencial de la cuenta de servicio con "Set up for use in HTTP Request node" y el scope `https://www.googleapis.com/auth/webmasters`.

**Correr a mano:** `POST /webhook/indexacion` con `{"brief_id": "BR-…", "url": "https://…"}`.

---

## Planilla

La misma del Scout (`SEO - Banco de keywords (Keyword Scout)`), compartida como **Editor** con la cuenta de servicio de n8n. Pestañas que usa el pipeline:

| Pestaña | Quién escribe | Columnas |
|---|---|---|
| `Keywords` | Scout; el Brief Agent completa `usado_en_brief` | ver el README del Scout |
| `Evidencia` | Scout | ver el README del Scout |
| `Briefs` | Brief Agent crea la fila; Writing Agent, Internal Linking, Portada y Auditor la actualizan por `id` | id, fecha, keyword_id, keyword, linea, cluster, keywords_secundarias, tipo_contenido, titulo, slug, estado, brief_json, investigacion_json, verificar, costo_usd, avisos, wp_post_id, publicar_el, borrador_url, palabras, marcadores, auditoria, enlaces_internos, enlaces_entrantes, publicado_el, portada_media_id, portada_url |
| `Casos` | Sebas, a mano | activo, rubro, situacion_inicial, que_hicimos, resultado, linea, se_puede_nombrar, nombre_o_ciudad |
| `Stickers` | Sebas, a mano | activo, archivo, que_es, temas |
| `Indexación` | SEO - Indexación; Sebas completa `pedido` | brief_id, url, publicado_el, sitemap_enviado, estado_google, ultimo_rastreo, ultima_revision, accion, pedido, pedido_el, notas |

**Estados de un brief:** `brief_listo` → `borrador` → `programado` o `bloqueado_auditoria` → `publicado`.

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
| Reglas de los enlaces internos | Internal Linking → `Code - Validar y aplicar` (`MAX`, `fraseValida`) y `Code - Armar pedido` (prompt, cantidad de candidatos) |
| Plantilla de la portada | Portada → `Code - Componer HTML` |
| Regla de tamaño del titular | Portada → `Code - Validar portada` (`TAMANOS`) |
| Reglas de los textos de la portada | Portada → `Code - Armar pedido` (prompt) y `Code - Validar portada` |
| Controles del Auditor | Auditor → `Code - Revisar posts` (bloqueos y avisos) y `Code - Decidir` (enlaces) |
| Plazos de la indexación (3 / 7 / 14 días) y hora de la revisión | Indexación → `Code - Resultados` y `Schedule - Diario 09:23` |
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
| Portada (prueba) | "Costo de agencia / de IA en 2026" · GUÍA PRÁCTICA · `monedas.png`. WebP 1600×900 subida como destacada (media 283) en 13 segundos. La primera corrida falló: DeepSeek se gastó los 4000 tokens razonando |
| Internal Linking (prueba en modo borrador) | 4 enlaces nuevos hacia posts publicados, USD 0,002. Dos anclas salieron flojas (una lista de herramientas): desde ahí el código rechaza anclas con listas separadas por comas |

Leyendo el 278 después de aprobado aparecieron problemas que las reglas no cubrían: dos secciones de preguntas frecuentes seguidas, el mismo dato (87,3%) repetido 5 veces, un ejemplo con montos inventados y una afirmación sin respaldo. Las dos secciones de preguntas y los datos repetidos se corrigieron en el Writing Agent y quedaron como controles del Auditor; el resto es lo que cubriría la revisión con IA.

---

## Trampas encontradas

| Síntoma | Causa | Cómo quedó resuelto |
|---|---|---|
| El brief llega con el JSON cortado | DeepSeek V4.1 Flash gasta `max_tokens` razonando antes de responder | `max_tokens: 16000` y `reasoning: { effort: 'low' }` |
| El editor devuelve el texto vacío | Mismo motivo: usó los 16000 tokens pensando | Editor con `reasoning: { enabled: false }` |
| La portada falla con "DeepSeek no devolvió JSON válido" | Mismo motivo, con un pedido de 3 líneas: 4000 tokens de razonamiento y `content: null` | `reasoning: { enabled: false }` en los pedidos cortos |
| El sticker PNG sale más chico que en la referencia | Los PNG traen margen transparente; el SVG de la referencia no | Recorte automático en el navegador antes de la captura |
| "01 // BLOG" y "pauta digital" salen en otra fuente | Chromium usa la de reserva sin avisar si un peso o tamaño no llegó a cargar | Fuente incrustada, `font-family` única y verificación de cada uso antes de capturar; si falla, no se captura |
| WordPress responde 503 al subir la portada en PNG (568 KB) o JPEG (248 KB) | El hosting no aguanta procesar esas imágenes (genera los tamaños intermedios); el WebP pasa sin problema | Portada en WebP. Cada intento fallido igual dejó un adjunto a medio crear en la biblioteca de medios |
| "El halo derecho casi no se ve" | El sticker tapa el centro del halo; medido contra la referencia, los píxeles visibles coinciden (±1) en PNG y en WebP | Nada que corregir en el formato; capas explícitas por las dudas |
| Sombra cyan doble en algunos stickers | Unos PNG la traen dibujada y otros no | Detección automática del borde cyan antes de agregarla |
| `Could not get parameter "columns.schema"` en el nodo de Sheets | La operación "agregar o actualizar" con mapeo automático necesita la lista de columnas declarada (la de "actualizar" no) | `columns.schema` con las columnas de la pestaña |
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

- [x] **Portada**: andando con 8 stickers (30 planeados).
- [ ] **2 imágenes en el cuerpo del artículo** con la misma estética (pendiente a pedido de Sebas, después de ver la portada funcionando con artículos reales).
- [x] **Indexación**: sitemap al publicar, revisión diaria con URL Inspection y lista del sábado para pedir a mano. IndexNow lo cubre Rank Math.
- [ ] **Revisión con IA en el Auditor** (propuesta, sin implementar): relleno, repeticiones, cifras sin fuente y afirmaciones flojas, con sugerencias por Telegram.
- [ ] **Largo**: los artículos salen ~35% más largos que lo pedido.
- [ ] **Casos**: cargar casos reales en la pestaña `Casos` (`activo` = `si`).
- [ ] **CTA de cursos**: hoy va a `/servicios/`; cambiar `CTA_URL` si se arma una página propia.
- [ ] **Keywords `refresh`**: el Scout las marca pero ningún workflow las trabaja todavía (actualizar el post existente en lugar de escribir otro).
- [ ] **Monitoreo post-publicación** con Search Console.
- [ ] **Volumen** con DataForSEO (etapa 1.5 del Scout).
- [ ] Opcional: asignar un error workflow en la configuración de cada workflow (no se puede por MCP).
