# Brief #1 — Agencia de automatización con IA para pymes

*29/10/2026 · Fuente: corridas KS-20261029-1208 y KS-20261029-1451 del [Keyword Scout](README.md) + análisis de Claude como orquestador.*

El volumen de búsqueda queda pendiente (DataForSEO, etapa 1.5): la prioridad sale de la intención de compra, la evidencia real y la debilidad de la SERP.

---

## 1. Keywords priorizadas

**Evidencia:** **AC** = Google Autocomplete Argentina · **X / R** = posts reales de X o Reddit de los últimos 30 días (los `E#` remiten a la sección 4). **SERP:** debilidad de 0 a 1 (más alto = más fácil) y quién rankea.

| # | Keyword | Línea | Intención | Evidencia | SERP |
|---|---|---|---|---|---|
| 1 | **agencia de automatización con IA para pymes** | Consultoría | 5 · contratar | AC + X/R (E1–E5, E8) | 0,6 · agenciaia.ai, automatizapyme, tecnobrain, reddit, suriscode |
| 2 | costo de agencia de IA / cuánto cuesta automatizar con IA | Consultoría | 5 · precio | X/R (E5, E6, E9, E17) | 0,6 |
| 3 | contratar agencia de IA para pymes | Consultoría | 5 · contratar | X + AC | 0,55 |
| 4 | agencia de sistematización y automatización con IA | Consultoría | 5 · contratar | AC | 0,55 · auttomai, wodes, automatizapyme |
| 5 | agente IA para inmobiliarias | Consultoría | 5 · contratar | AC | 0,65 · dapta, youtube, rendair, tec5 |
| 6 | formar equipo interno de IA / agencia o equipo interno | Consultoría | 4 · comparación | X (E1, E4) | 0,55 |
| 7 | cómo elegir una agencia de IA / señales de estafa | Consultoría | 4 · confianza | X (E2, E5, E8) | sin consultar |
| 8 | implementar IA en pymes / por dónde empezar | Consultoría | 4 · cómo empezar | X/R (E11, E12) | 0,5 |
| 9 | mejor IA para inmobiliarias | Consultoría | 4 · comparación | AC + pregunta AC | 0,55 |
| 10 | agente de IA para WhatsApp (pymes) | Consultoría | 4 · contratar | R (E15, E19, E20) | sin consultar |
| 11 | consultoría en IA para pymes | Consultoría | 4 · contratar | R (E14, E16) | difícil · Globant, EY, IBM |
| 12 | n8n vs make vs zapier (precio, español) | Cursos | 4 · comparación | AC (6 variantes) + R (E25, E33, E34) | 0,55–0,9 |
| 13 | Zapier caro: alternativas | Cursos | 4 · alternativas | R (E33) | sin consultar |
| 14 | curso de n8n desde cero / gratis en español | Cursos | 4 · curso | R + AC | 0,6 · youtube, coderhouse, udemy, platzi |
| 15 | aprender automatización con IA | Cursos | 4 · curso | R + AC | 0,6 |
| 16 | curso de automatización con n8n | Cursos | 4 · curso | R + AC | 0,5 |
| 17 | curso de automatización con IA (gratis) | Cursos | 4 · curso | AC | 0,45 · Santander, UNC, Coderhouse |
| 18 | curso IA para inmobiliarias | Cursos / consultoría | 4 · curso | AC | 0,6 |
| 19 | Claude + n8n para crear workflows | Cursos | 3 · cómo hacer | R (E25, E26, E28, E36) | **refresh** de los posts propios de Claude Code y de agentes IA |
| 20 | ¿hace falta saber programar para n8n? | Cursos | 3 · problema | R (E24, E32) | sin consultar |
| 21 | cuánto se tarda en aprender n8n | Cursos | 3 · cómo empezar | R (E27) | sin consultar |
| 22 | qué es un agente de IA | Cursos | 2 · informacional | AC (muchas variantes) | 0,28–0,4 · Google Cloud, IBM, Xataka |

**Clusters por intención:**
- Contratar o evaluar ayuda: 1, 2, 3, 4, 6, 7, 11.
- Verticales: 5, 9, 10, 18.
- Empezar a implementar: 8.
- Comparar herramientas: 12, 13.
- Aprender y cursos: 14–17, 19–21.
- Tope del embudo: 22.

**Descartadas:** "vender automatizaciones a pymes / conseguir clientes" (E17, E35). Hay mucha conversación, pero es gente que quiere *armar* una agencia, no el comprador de Pauta Digital.

---

## 2. Brief — listo para el Writing Agent

**Keyword principal:** agencia de automatización con IA para pymes

**Keywords secundarias:**
- costo de agencia de IA
- contratar agencia de IA para pymes
- agencia de sistematización y automatización con IA
- formar equipo interno de IA
- consultoría en IA para pymes

**Search intent:** comercial-investigativo. El dueño de una pyme compara proveedores antes de contratar, con intención transaccional cerca (precio y "quién me lo hace").

**Tipo de contenido:** guía de decisión del lado del comprador: checklist + comparación + rangos de costo. 2.200–2.600 palabras.

**Datos para Rank Math:**
- **Título:** Agencia de automatización con IA para pymes: guía real en 7 preguntas
- **Slug:** `agencia-automatizacion-ia-pymes`
- **Meta description** (140 caracteres): Agencia de automatización con IA para pymes: cuánto cuesta, qué tiene que entregarte y cuándo no te conviene contratarla. 7 preguntas clave.

### Ángulo diferenciador

Lo escribe alguien que vende consultoría y aun así te dice **cuándo no contratar a nadie**. En X se repite la misma crítica (E1, E2): con una suscripción de USD 200 cualquiera se llama "agencia de IA", y si tu proveedor te necesita para siempre, es una consultora con otra etiqueta.

Encaja directo con la home ("Menos proveedores. Más sistema."). El criterio que ordena el artículo es la **independencia**: que el sistema, las cuentas y los flujos queden a nombre del cliente.

### Estructura

- **H1:** el título.
- **Intro (~400 palabras):** keyword exacta en la primera oración. Hook con la señal real del mercado: agencias de IA que aparecen de un día para el otro (E1, E2). Promesa: 7 preguntas para no pagar humo.
- **H2 — Qué hace (y qué no) una agencia de automatización con IA para pymes**
  - H3: Implementación, consultoría o curso: tres modelos distintos
  - H3: La prueba de independencia: ¿el sistema queda a tu nombre?
- **H2 — Cuánto cuesta una agencia de automatización con IA**
  - H3: Setup + abono mensual: cómo leer una cotización. Ejemplos públicos: USD 5.000 de setup + USD 1.500/mes (E5) y USD 1.000 + USD 250/mes (E17). Presentarlos como casos, no como promedio de mercado.
  - H3: Costos que no aparecen en el presupuesto: APIs, licencias y mantenimiento (E6, E7, E9)
  - H3: `[CASO: cómo cotiza Pauta Digital después del diagnóstico]`
- **H2 — 7 preguntas para hacerle a una agencia de IA antes de contratarla** (el checklist central)
  1. ¿Qué problema de negocio resuelve, más allá del chatbot? (E12)
  2. ¿De quién son las cuentas, credenciales y flujos? (E5: cuenta white-label)
  3. ¿Qué pasa si dejo de pagar el abono?
  4. ¿Quién mantiene el sistema cuando algo se rompe? (E7)
  5. ¿Cómo vamos a medir el resultado? (E16)
  6. ¿El diagnóstico es pago y cuánto cuesta? (E8: USD 50.000 "solo para explorar")
  7. ¿Me capacitan para no depender de ustedes? (E1)
- **H2 — Señales de alerta: cuándo una agencia de automatización con IA es humo**
- **H2 — ¿Agencia, consultor o alguien interno? Cómo decidir según tu pyme**
  - H3: Tabla comparativa: costo, velocidad, dependencia, conocimiento que queda (E4)
- **H2 — Cómo lo trabajamos en Pauta Digital: diagnóstico → roadmap → implementación** `[CASO]`
- **H2 — Preguntas frecuentes**
- **Conclusión + CTA**

### Preguntas que tiene que responder

**De conversaciones reales:**
- ¿Contrato una agencia de IA o formo a mi equipo? (E1)
- ¿Cuánto cobra una agencia de IA? (E5, E17)
- ¿Vale la pena para una pyme? (E4)
- ¿Por qué la factura de IA sale más cara de lo esperado? (E6, E9)
- ¿Quién mantiene las automatizaciones? (E7)
- ¿Qué hace realmente un consultor de IA? (E16)
- ¿Necesito un chatbot o mi problema es otro? (E12)

**Fan-out para IA** (inferidas: Serper no devuelve People Also Ask para Google Argentina):
- ¿Qué conviene automatizar primero en una pyme?
- ¿En cuánto tiempo está funcionando?
- ¿Qué herramientas usan (n8n, Make, Zapier)?
- ¿Necesito saber programar?
- ¿Es seguro darles acceso a mis cuentas y datos?
- ¿Se puede automatizar WhatsApp?
- ¿Qué retorno puedo esperar?

### Entidades a mencionar

- **Herramientas:** n8n, Make, Zapier, Claude, ChatGPT, agentes de IA, WhatsApp Business API, CRM, GoHighLevel (white-label), Meta Ads, Google Ads, GA4, Looker Studio, MCP.
- **Conceptos:** setup fee, abono mensual, dependencia del proveedor (vendor lock-in), mantenimiento, discovery, ROI, diagnóstico Pareto, roadmap.
- **Competidores en la SERP:** agenciaia.ai, automatizapyme.com, tecnobrain.ar, suriscode.com.

### Gaps de los resultados actuales

- El top está formado por páginas de servicio de agencias que se venden a sí mismas, listicles donde el autor se ubica primero (suriscode) y contenido para quien quiere *crear* una agencia.
- Nadie escribe del lado del comprador: costos reales, checklist, señales de alerta y "cuándo no contratar".
- Que Reddit aparezca en el top 5 indica demanda de respuestas honestas.
- Límite del análisis: se vieron dominios y títulos, no el contenido completo de esas páginas.

### CTA

Diagnóstico gratis de 20 minutos, el mismo de la home.
- **A mitad del artículo**, después del checklist: "Si querés, revisamos tu caso con estas 7 preguntas".
- **Al cierre:** `[LINK: contacto]`.
- **Tipo de CTA en el pipeline SEO:** `consultoria`.

### Notas para GEO / citas de IA

- Una definición de una oración al principio: "Una agencia de automatización con IA para pymes es…".
- Cada H2 arranca con la respuesta directa en 40–60 palabras.
- El checklist numerado y la tabla comparativa son los formatos más citables.
- FAQ con schema `FAQPage` en Rank Math.
- Cifras siempre con fuente y fecha. **Verificar antes de publicar** el estudio que cita Forbes (E6: 49 % recibió una factura más alta que la esperada).
- Firma con la experiencia concreta de Sebas y fecha de actualización visible.

### Reglas de voz

- Voseo rioplatense.
- "Anuncios" o "campañas", nunca "ads".
- Prohibido el patrón "Eso no es X. Es Y.".
- Cero cifras inventadas.
- Los casos propios van marcados como `[CASO: …]`.

### Línea para el bot del pipeline SEO

```
guía | agencia de automatización con IA para pymes | Ángulo comprador: cuándo NO contratar, prueba de independencia (sistema y cuentas propias), 7 preguntas, costos setup + abono con ejemplos públicos, agencia vs consultor vs interno. CTA diagnóstico gratis.
```

---

## 3. Keywords para los próximos briefs

1. **#2 + #7**: un post de "cuánto cuesta y cómo no caer en humo" (puede ir junto).
2. **#5 + #9**: IA y agentes para inmobiliarias, la vertical foco.
3. **#12 + #13**: n8n vs Make vs Zapier y alternativas a Zapier.
4. **#14 + #15 + #21**: aprender n8n / automatización desde cero.
5. **#10**: agentes de IA para WhatsApp.
6. **#8**: implementar IA en una pyme, por dónde empezar.
7. **#19**: Claude + n8n, como **refresh** de los posts existentes, no como artículo nuevo.

---

## 4. Evidencia citada

Posts verificados por el Keyword Scout (la URL figuraba entre las fuentes que Grok visitó). Los textos son extractos.

| Id | Fuente | Fecha | Extracto |
|---|---|---|---|
| E1 | [X](https://x.com/betobetico/status/2114681102019224014) | 26/10/2026 | "Si tu proveedor te necesita para siempre, es una consultora con otra etiqueta. ¿Contratarías una o formarías a tu equipo?" |
| E2 | [X](https://x.com/mardehaym/status/2114644751605530674) | 26/10/2026 | "Anyone with a $200 subscription can call themselves an AI agency." |
| E3 | [X](https://x.com/mardehaym/status/2112922121169170829) | 21/10/2026 | "I told a company not to hire us last week." |
| E4 | [X](https://x.com/girdley/status/2110187157126062429) | 14/10/2026 | "I know ~35 SMB CEOs. Only one of them was looking for an AI agency…" (175 likes, 73 respuestas) |
| E5 | [X](https://x.com/lukepierceops/status/2112668211539841498) | 20/10/2026 | Auditoría de un sistema "done for you": white-label de GoHighLevel, USD 5K de setup + USD 1.500/mes |
| E6 | [X](https://x.com/Forbes/status/2114144915248820641) | 25/10/2026 | "49% of small business AI buyers received a bill higher than they expected" (verificar el estudio original) |
| E7 | [X](https://x.com/bradsmith/status/2110831962671767707) | 15/10/2026 | "Marketing agencies are about to learn how much maintenance automation needs" |
| E8 | [X](https://x.com/mardehaym/status/2115484094171644399) | 28/10/2026 | "A project shop quoted… $50,000 just to explore" |
| E9 | [X](https://x.com/karmahvhe/status/2115641397621485874) | 29/10/2026 | "Hidden costs in AI tooling need urgent attention" |
| E11 | [X](https://x.com/UnlockSmallBiz/status/2111811826874773926) | 18/10/2026 | "Not enough SMB owners showing what they've actually implemented" |
| E12 | [Reddit](https://www.reddit.com/r/AiAutomations/comments/1xbptck/title_i_run_an_ai_development_company_and_ive_realised/) | 21/10/2026 | "Someone wants an AI chatbot, but their actual problem is that customer enquiries are getting lost." |
| E14 | [Reddit](https://www.reddit.com/r/AI_Agents/comments/1x441em/we_compared_9_enterprise_ai_consulting_firms_for/) | 12/10/2026 | Comparación de 9 consultoras de IA |
| E15 | [Reddit](https://www.reddit.com/r/SaaS/comments/1xg95jf/are_ai_receptionists_still_a_good_business/) | 26/10/2026 | Agentes de IA que atienden conversaciones de punta a punta |
| E16 | [Reddit](https://www.reddit.com/r/N_iX/comments/1x0q05d/what_ai_consultants_actually_do_and_where_most/) | 08/10/2026 | "What AI Consultants Actually Do — and Where Most Engagements Fall Short" |
| E17 | [Reddit](https://www.reddit.com/r/AiAutomations) | 29/10/2026 | Primer cliente de una agencia: USD 1.000 de setup + USD 250/mes (la URL apunta al subreddit, no al post) |
| E19 | [Reddit](https://www.reddit.com/r/aiagents/comments/1xckpdx/ai_agent_agenciesdo_you_resell_the_platform_or/) | 22/10/2026 | Agentes de WhatsApp: ¿revender la plataforma o cobrar solo el armado? |
| E20 | [Reddit](https://www.reddit.com/r/AI_Agents/comments/1xfw123/the_booking_bug_that_taught_us_to_never_let_the/) | 26/10/2026 | Agentes de WhatsApp y teléfono para pymes (gimnasios, spas, clínicas) |
| E24 | [Reddit](https://www.reddit.com/r/n8n/comments/1xf9jrb/n8n_learned_now_what_what_technical_skills_are/) | 25/10/2026 | "n8n learned, now what?" |
| E25 | [Reddit](https://www.reddit.com/r/n8n/comments/1xc21kj/nobody_use_n8n_make_dify/) | 21/10/2026 | "Claude is good for writing the individual nodes but n8n still handles the orchestration" |
| E26 | [Reddit](https://www.reddit.com/r/n8n/comments/1x98gu5/how_to_claude_reposkillsmcp/) | 18/10/2026 | Claude + skills + MCP para automatizar |
| E27 | [Reddit](https://www.reddit.com/r/n8n/comments/1xbmk6t/how_much_do_i_need_to_learn_n8n_i_dont_know/) | 21/10/2026 | "How much do I need to learn n8n?" |
| E28 | [Reddit](https://www.reddit.com/r/n8n/comments/1xb5k2h/how_do_you_get_claude_code_to_build_complex_n8n/) | 20/10/2026 | Claude Code + MCP de n8n para workflows complejos |
| E32 | [Reddit](https://www.reddit.com/r/automation/comments/1xe5f2d/do_you_really_need_to_know_code_to_build/) | 24/10/2026 | "Do you really need to know code to build automations?" |
| E33 | [Reddit](https://www.reddit.com/r/BusinessAutomationHub/comments/1xdwjy3/zapier_is_getting_expensive_when_switching_tools/) | 23/10/2026 | "Zapier is getting expensive" |
| E34 | [Reddit](https://www.reddit.com/r/n8n/comments/1xe51v2/which_automation_tool_actually_scales_best_beyond/) | 24/10/2026 | n8n vs Make vs Relay |
| E36 | [Reddit](https://www.reddit.com/r/n8n/comments/1w44pgj/comment/p779zq0/) | 01/09/2026 | "Claude as the builder and n8n as the executor" |
