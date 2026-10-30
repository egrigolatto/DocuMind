# Registro de decisiones — DocuMind

> Última actualización: 2026-10-29

Convención de estados: **Confirmado** (aprobado por el usuario) · **Propuesto** (recomendación pendiente de aprobación) · **Pendiente** (sin resolver) · **Superado** (reemplazado por otra decisión, indicar cuál).

Este registro es un documento vivo: se actualiza durante todo el desarrollo, también cuando la implementación se desvíe del diseño original.

## Decisiones

### D-01 — Monolito modular en una aplicación Next.js
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: MVP de una sola persona con doble objetivo (portfolio + aprendizaje); no hay requisitos de escalar ni de equipos paralelos.
- **Decisión**: Un único proyecto Next.js (App Router) con módulos internos por responsabilidad: UI, HTTP, casos de uso, documentos, RAG, IA y datos.
- **Alternativas evaluadas**: Microservicios o servicios separados (API + worker) — descartados por el propio owner como fuera de alcance; SPA + API Express aparte — descartada: dos proyectos donde basta uno.
- **Consecuencias**: Regla de dependencias unidireccional (`app → modules → ai|db`); un solo `pnpm dev`; despliegue trivial. El crecimiento real de carga justificaría separar ingesta en el futuro.

### D-02 — Runtime Node en el servidor (no Edge)
- **Fecha**: 2026-10-29
- **Estado**: Confirmado (consecuencia técnica de D-01)
- **Contexto**: El parsing de PDF y el driver `pg` requieren APIs de Node.
- **Decisión**: Toda la aplicación corre con runtime Node de Next.
- **Alternativas evaluadas**: Runtime Edge — descartado: sin soporte para estas cargas de trabajo.
- **Consecuencias**: Despliegue en cualquier host con Node/Docker; sin límites de Edge runtime que aprender a esquivar.

### D-03 — Pipeline RAG completo del lado del server; UI capa fina
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: La clave API de OpenRouter no puede llegar al navegador (RNF-08) y los datos viven en Postgres.
- **Decisión**: Extracción, chunking, embeddings, búsqueda y generación ocurren en código de servidor; el cliente solo renderiza.
- **Alternativas evaluadas**: Ninguna razonable dado D-02/RNF-08.
- **Consecuencias**: El cliente es tonto y testeable visualmente; toda la lógica de aprendizaje vive en `src/modules/`.

### D-04 — Capa HTTP con route handlers REST en lugar de server actions
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: Se pidió explícitamente mantener separadas las responsabilidades de interfaz, manejo HTTP, casos de uso, procesamiento, embeddings, IA y datos (pedido de alcance del owner, reflejado en D-01 y RNF-01).
- **Decisión**: Endpoints `POST/GET /api/documents`, `DELETE /api/documents/[id]`, `POST /api/chat`; los handlers delegan a casos de uso y no contienen lógica de negocio.
- **Alternativas evaluadas**: Server actions — descartadas para el MVP: más rápidas de escribir pero difuminan la frontera HTTP/caso de uso, que es parte del objetivo didáctico. API externa no: no hay clientes de terceros.
- **Consecuencias**: La API es documentable y demostrable (curl en README futuro); pequeño costo de boilerplate.

### D-05 — PostgreSQL + pgvector como almacén único (datos relacionales + vectores)
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: El owner ya conoce Postgres y eligió pgvector en su stack; el volumen del MVP es pequeño.
- **Decisión**: Documentos, chunks y embeddings viven en la misma base; sin almacén vectorial dedicado.
- **Alternativas evaluadas**: Qdrant/Pinecone/Chroma — descartadas: una tecnología nueva más sin necesidad de escala; SQLite-vec — descartada: fuera del stack elegido por el usuario.
- **Consecuencias**: Una sola migration story y un solo container; integridad referencial real (cascade delete de vectores); los índices vectoriales de pgvector son más simples que los de un dedicado — suficiente aquí y didáctico (F5).

### D-06 — Sin frameworks RAG ni SDKs: `fetch` directo a OpenRouter validado con Zod
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: Objetivo explícito del proyecto: implementar y comprender el flujo fundamental en TypeScript.
- **Decisión**: Cliente HTTP propio en `src/modules/ai/client.ts` (fetch + tipos + reintentos + backoff), respuestas validadas con Zod.
- **Alternativas evaluadas**: LangChain/LlamaIndex — descartados por decisión del owner (ocultan el flujo); SDK `openai` apuntando a OpenRouter — descartado: cómoda pero oculta el shape real de la API.
- **Consecuencias**: Hay que escribir parseo, manejo de errores y batching a mano (trabajo extra = aprendizaje deseado); los tipos propios son documentación viva del protocolo.

### D-07 — Modelo de datos mínimo: dos tablas; el texto extraído se persiste solo en chunks
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: Sin auth ni historial persistente, la superficie de datos del MVP es documentos + trozos vectorizados.
- **Decisión**: `documents` (metadatos, estado, trazabilidad de embeddings) y `chunks` (texto + `vector(N)` + posición). Sin tabla intermedia de texto completo y **sin persistir el binario original del archivo**: la extracción ocurre en la request de ingesta y lo único que queda en la base son los chunks.
- **Alternativas evaluadas**: Guardar el texto completo además en `documents` — descartado por ahora: los chunks con posición permiten reconstruir el orden; si en F7 surge la necesidad (exportar/previsualizar), se agrega con migración limpia. Guardar el PDF en disco o en `bytea` — derivado a P-11 para no decidir el "cómo" sin necesidad: hoy re-procesar un documento significa volver a subirlo.
- **Consecuencias**: Eliminar en cascada es trivial; **re-chunking o cambio de modelo de embeddings exigen re-subir los archivos** (no hay binarios ni texto fuente para reprocesar). Es el costo de mantener el esquema en dos tablas; T8.4 y P-11 lo reevalúan si el reprocesamiento se vuelve necesario.

### D-08 — Single-tenant: sin autenticación ni `user_id`
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: El owner excluyó auth del MVP; la app es una demo local/portafolio.
- **Decisión**: Ninguna tabla tiene `user_id`; todos los visitantes de una instancia comparten el índice.
- **Alternativas evaluadas**: Auth con NextAuth — descartada por alcance; un usuario "duro" — descartada: abstracción sin necesidad concreta.
- **Consecuencias**: Si mañana se agrega multiusuario, es una migración con backfill simple pero transversal; queda registrado aquí para que no sea sorpresa. Riesgo asociado documentado: no exponer la demo a internet sin revisar esto.

### D-09 — Ingesta síncrona dentro de la request, con límite de 5 MB por documento
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: Sin colas ni workers en alcance; los documentos del MVP (PDFs de pocas decenas de páginas, TXT) se procesan en segundos.
- **Decisión**: `POST /api/documents` extrae, hace chunking, genera embeddings y deja el documento en `ready` o `error` dentro de la misma petición. Límite de tamaño validado con Zod antes de procesar (valor inicial 5 MB, ajustable, P-08).
- **Alternativas evaluadas**: Jobs de fondo con tabla de cola — descartada por alcance (el owner excluyó procesamiento distribuido); la alternativa mínima (fire-and-forget sin transacción) tiene peor manejo de errores que el modo síncrono para este tamaño.
- **Consecuencias**: UX simple (esperás, ves el estado); timeout de request como límite real — se documenta y se mide en F4; la regla de integridad es que **cualquier paso que falle deja el documento en `error` con `error_message` y sin chunks huérfanos** (la inserción de chunks + embeddings corre en una transacción; ver RF-09/HU-06); un futuro `POST /api/documents/[id]/reprocess` cubre el caso de documentos grandes (subordinado a P-11: exige conservar el original).

### D-10 — Modelo de embeddings: `nvidia/nemotron-3.5-embed-1b:free` vía OpenRouter, con trazabilidad por documento
- **Fecha**: 2026-10-29
- **Estado**: Confirmado (decisión del owner tras presentar alternativas)
- **Contexto**: Se necesitó elegir entre costo cero y estabilidad. Verificado 2026-10-29: OpenRouter expone `/api/v1/embeddings` con 38 modelos; la lista de la API no publica dimensiones. Nota verificada post-aprobación: el modelo elegido es **multimodal** (`text+image → embedding`, diseñado también para páginas de documento como imagen); DocuMind explota **solo su modalidad texto**.
- **Decisión**: Usar el modelo gratuito (32k tokens de contexto, costo 0) solo para texto. El nombre se configura por variable de entorno; cada documento persiste `embedding_model` y `embedding_dims`.
- **Alternativas evaluadas**: En la decisión original (2026-10-29) se compararon: `openai/text-embedding-3-small` (~$0.02/M tokens, 1536 dims estables y estándar de industria), `perplexity/pplx-embed-v1-0.6b` (~$0.004/M) y `voyageai/voyage-4-lite` — las tres de pago, descartadas por preferencia del owner de presupuesto cero; el free-tier elegido quedó como única opción sin costo razonable para el proyecto. Otros modelos `:free` del catálogo que **no se consideraron en ese momento** y se registran aquí en revisión: `nvidia/nemotron-3.5-embed-8b:free` (misma familia, 8B: probable mayor calidad, también mayor latencia bajo free-tier — candidato natural si la 1B demuestra recuperación pobre), `nvidia/nemotron-3-embed-1b:free` (generación anterior), `liquid/lfm-2.5-embedding-350m:free` (descartable: contexto 512 insuficiente para chunks de documento). Si P-01 o los tests de F3 muestran problemas, la variable `OPENROUTER_EMBEDDINGS_MODEL` permite cambiar sin tocar código.
- **Consecuencias**: Riesgos asumidos y mitigados: dimensiones desconocidas → probe obligatorio antes de fijar `vector(N)` (P-01); rate limits no garantizados del free-tier → batching + backoff en el cliente; cambio de modelo futuro → reindexación completa (las columnas de trazabilidad lo hacen detectable y migrable). La modalidad imagen queda deliberadamente sin usar: no confunde con el OCR excluido (RNF-06) porque el pipeline nunca envía imágenes, solo texto ya extraído.

### D-11 — Extracción PDF con `unpdf`
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: Se requiere extraer texto de PDFs con texto seleccionable en runtime Node, sin configurar un navegador headless.
- **Decisión**: `unpdf` (v2.0.1 verificada 2026-10-29; construida sobre pdf.js, orientada a servidor, API mínima `getText`).
- **Alternativas evaluadas**: `pdf-parse` v2.4.5 — reescrita hace poco, API por verificar (segunda opción si unpdf falla en F2); `pdfjs-dist` directo — máximo control pero más boilerplate del necesario; plugin `file-parser` de OpenRouter — descartado: delega la extracción (el objetivo es implementarla localmente); Textract/externalidades — fuera de alcance.
- **Consecuencias**: Decisión a validar con PDFs reales en F2; si la calidad de extracción es insuficiente para algún caso, se reevalúa con datos (estado pasaría a Superado con nueva entrada).

### D-12 — Historial del chat efímero (solo en cliente)
- **Fecha**: 2026-10-29
- **Estado**: Confirmado (decisión del owner)
- **Contexto**: Sin auth, persistir conversaciones mezclaría los chats de todos los visitantes anónimos en un mismo historial compartido.
- **Decisión**: Los mensajes viven en el estado del navegador; solo el refresh los pierde. La base guarda únicamente documentos y chunks.
- **Alternativas evaluadas**: Tablas `conversations`/`messages` — descartada para el MVP: mejor para portfolio a largo plazo pero inconsistente con D-08; sessionStorage/localStorage — no aporta en MVP.
- **Consecuencias**: Menos esquema y menos UI; reevaluar en F8 junto con auth si se quiere demo "asistente con memoria".

### D-13 — Streaming SSE fuera del MVP
- **Fecha**: 2026-10-29
- **Estado**: Confirmado (decisión del owner)
- **Contexto**: F6 es la fase crítica del pipeline; agregar streaming complica parseo de fuentes y manejo de errores.
- **Decisión**: Respuesta completa no-streaming en el MVP; SSE queda en F8.
- **Alternativas evaluadas**: Streaming desde F6 — descartada: UX de demo mejor pero más superficie de fallo en la fase más densa.
- **Consecuencias**: Latencias percibidas mayores (respuestas de hasta ~20 s, RNF-10); el incremento de F8 es acotado porque el cliente ya aísla el call al LLM.

### D-14 — Set de documentación propio: cinco documentos + TASKS.md, brief fusionado en README
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: El owner definió sus documentos objetivo en español, incluyendo `docs/learning-notes.md` para el objetivo de aprendizaje.
- **Decisión**: `README.md` (incluye visión/alcance, sin project-brief aparte), `docs/requirements.md`, `docs/architecture.md` (equivalente al diseño técnico), `docs/decisions.md`, `docs/learning-notes.md`, `TASKS.md`.
- **Alternativas evaluadas**: `docs/project-brief.md` de la plantilla estándar — descartado: duplicaría el README para un proyecto de este tamaño.
- **Consecuencias**: Nomenclatura propia; el README enlaza y no duplica; `learning-notes.md` se alimenta al cerrar cada fase.

### D-15 — Registro de uso y costo de OpenRouter en cada llamada
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: El objetivo de control de costos (RNF-07) necesita visibilidad real por request.
- **Decisión**: El cliente de IA persiste en log estructurado el `usage` (tokens) y el `cost` que devuelve OpenRouter en cada respuesta.
- **Alternativas evaluadas**: Consultar el dashboard de OpenRouter manualmente — descartada: no deja datos dentro del proyecto; contadores propios estimados — descartada: el proveedor ya devuelve cifras nativas.
- **Consecuencias**: El log permite sumar costos por fase y auditar el free-tier; una feature "cuánto costó indexar este documento" requeriría volver ese log consultable por documento (columna o tabla `ai_calls`) — se decide solo si la feature se pide, hoy el log alcanza para RNF-07.

### D-16 — Proceso: Conventional Commits, ramas por fase/tarea, nada automático
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: El owner pidió buenas prácticas profesionales desde el inicio y vetó commits u operaciones destructivas automáticas.
- **Decisión**: `main` + ramas `feat/fN-descripcion` o `docs/...`; mensajes bajo Conventional Commits; cada commit lo ejecuta el usuario tras revisar los cambios propuestos.
- **Alternativas evaluadas**: Commit por cada tarea sin rama — descartada: dificulta retroceder por fase; trunk-based con CI — se considera en F8.
- **Consecuencias**: Historial legible del proceso de aprendizaje (útil en portfolio); el agente nunca corre `git commit` sin pedido explícito.

### D-17 — Inyección de prompts: riesgo aceptado y documentado en el MVP
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: El contenido de los documentos pasa textualmente al prompt del LLM; un documento malicioso podría intentar instruir al modelo. Analizado en la planificación (riesgo R-08 de architecture.md) y aprobado como parte del package.
- **Decisión**: Para el MVP local single-tenant (D-08), el riesgo se acepta con dos mitigaciones: (a) system prompt defensivo en F6 (las instrucciones del usuario-no-documento tienen precedencia; el contexto se delimita con separadores), (b) demostración y registro del vector de ataque en F7 (T7.3) y learning-notes. No se implementa filtrado de contenido ni sandboxing de prompts.
- **Alternativas evaluadas**: Sanitizado agresivo del texto de chunks — descartado: rompe el contenido legítimo y su efectividad es limitada; ignorar el tema sin documentación — descartado: el portfolio exige conocer los límites de lo que se presenta.
- **Consecuencias**: Si la app llegara a internet con uploads ajenos (fuera del MVP), esta decisión queda **Superada** y se requiere re-diseño (separar instrucciones de datos de forma más estricta).

### D-18 — Elecciones de librerías de soporte: Drizzle, Vitest (+vite), Tailwind, tests colocados
- **Fecha**: 2026-10-29
- **Estado**: Confirmado
- **Contexto**: Decisiones de stack del owner que se registran con sus alternativas para completar el trazabilidad del registro (el análisis original las aprobó como package P1-P5).
- **Decisión y alternativas descartadas**:
  - **ORM**: Drizzle — esquema tipado + migraciones SQL explícitas y legibles (didáctico). Descartados: Prisma (magia de engine y migraciones binarias, más lejos del SQL), Kysely (solo query-builder, sin kit de migraciones tan cómodo), SQL puro con `pg` (máximo aprendizaje de SQL pero plantillas string sin tipos: más errores en un pipeline ya de por sí complejo).
  - **Runner de tests**: Vitest — integración nativa con el ecosistema Vite/TS del proyecto, ya elegido por el owner. Nota técnica (revisión 2026-10-29): Vitest 5 exige `vite ^6 || ^7 || ^8` como peer: se instala como devDependency en T0.5. Descartados: Jest (config más pesada para ESM/TS), `node:test` (sin watch/coverage maduros, menos familiar).
  - **Estilos**: Tailwind CSS 4 — viene en el scaffold de Next y alcanza para una UI de una pantalla. Descartado: CSS Modules puro (más archivos para un diseño simple), shadcn/ui (dependencia de más abstracción de componentes fuera del objetivo de aprendizaje).
  - **Gestor**: pnpm — elegido por el owner; lockfile reproducible y espacio eficiente. Descartados: npm (más lento, sin catálogo), Bun (runtime nuevo; el objetivo es aprender RAG, no otro runtime).
  - **Ubicación de tests**: junto al código (`*.test.ts`) — visibilidad inmediata de qué está testeado; se separa solo si crece. Estado: parte del package P3 aprobado.
- **Consecuencias**: Estas cuatro elecciones son de bajo riesgo y reversible estándar; ninguna introduce una abstracción que oculte el flujo RAG.

### D-19 — Estructura de carpetas por dominio bajo `src/modules/`
- **Fecha**: 2026-10-29
- **Estado**: Confirmado (package P3 de la propuesta aprobada el 2026-10-29)
- **Contexto**: El owner pidió organización clara y proporcional, sin carpetas especulativas, con las responsabilidades del punto 4 separadas.
- **Decisión**: El árbol publicado en `docs/architecture.md#estructura-de-carpetas` (parcial, a materializar en F0): `src/app` (HTTP/UI-shell), `src/components`, `src/modules/{documents,rag,ai,db}`, `src/lib`, más `scripts/` para artefactos desechables (probe de dimensiones, carga de demo).
- **Alternativas evaluadas**: Organización por capa técnica (`controllers/`, `services/`, `repositories/`) — descartada: fragmenta el pipeline RAG, que es la unidad conceptual que se quiere enseñar; estructura aún más plana (`lib/` único) — descartada: el crecimiento del pipeline haría crecer un cajón de sastre.
- **Consecuencias**: Un módulo por etapa del RAG = la estructura del repo es un mapa del flujo. Si en implementación una carpeta no justifica su existencia, se colapsa y se registra como desviación.

### D-20 — Plan por fases F0-F8 con desglose tardío
- **Fecha**: 2026-10-29
- **Estado**: Confirmado (package P4/P5 de la propuesta aprobada el 2026-10-29)
- **Contexto**: Metodología de aprendizaje progresivo del owner: una tarea por vez, aprobada, verificada; sin generar toda la aplicación de una vez.
- **Decisión**: Ocho fases ordenadas por dependencias (entorno → datos → extracción → embeddings → ingesta → retrieval → generación → robustez), desglosadas en detalle solo F0/F1; el resto se desglosa al comenzar (TASKS.md).
- **Alternativas evaluadas**: Desglose completo desde el día 1 — envejece mal y contradice la exploración progresiva; desarrollo vertical (un RAG mínimo end-to-end en semanas 1-2) — tentador pero mezcla cinco conceptos nuevos de golpe, contra el objetivo de comprensión.
- **Consecuencias**: Cada fase produce algo verificable y una entrada en learning-notes; los pendientes técnicos (P-01…P-12) tienen fecha y fase de resolución asignada.

## Cuestiones pendientes

| ID | Tema | Qué falta para resolverlo |
| --- | --- | --- |
| P-01 | Dimensiones reales del embedding `nvidia/nemotron-3.5-embed-1b:free` | Script probe contra `/api/v1/embeddings` con la API key del owner (primera tarea de F1, antes de fijar `vector(N)` en la migración) |
| P-02 | Compatibilidad de TypeScript 7.0.2 con Next 16.4 y drizzle-kit 0.31 | Verificación en F0 contra el scaffold de create-next-app; registrar versión final fijada como nueva entrada D-xx |
| P-03 | Tag vigente de la imagen Docker con pgvector incluido (`pgvector/pgvector:pg17` u otro) | Consulta a Docker Hub en F0; probar `docker compose up` y extensión disponible |
| P-04 | Modelo default de chat en OpenRouter | Decidir en F6 con experimentos reales de calidad/costo sobre prompts típicos del MVP |
| P-05 | Tamaño de chunk y porcentaje de overlap | Experimentos en F2 con documentos reales y métricas simples (chunks generados, tokens por chunk) |
| P-06 | Valor de k y umbral de similitud para "sin información relevante" | Mediciones en F5/F6 con preguntas preparadas contra PDFs de prueba |
| P-07 | Licencia del repositorio | Elección del owner (recomendación pendiente: MIT o 0BSD para portfolio público; no se decide por él) |
| P-08 | Límite de body multipart real del runtime Next 16 y ajuste del límite de 5 MB | Prueba de upload en F4 con archivos borde |
| P-09 | Índice vectorial: HNSW vs IVFFlat, parámetros | Decidir en F5 con datos y explicación de trade-offs |
| P-10 | Tipo de PK en `documents`/`chunks`: uuid vs serial. **Propuesto: uuid** (IDs opacos y estables para citas en UI y rutas `/api/documents/[id]`) | Fijarlo en T1.2 antes de la primera migración |
| P-11 | Estrategia de persistencia del binario original (disco local vs `bytea` vs no persistir — estado actual D-07) condiciona T8.4 (reprocess) y T7.5 | Decidirla recién si surge la necesidad concreta (F7/F8); no crear infra por las dudas |
| P-12 | Mecanismo de actualización del listado de documentos: **Propuesto** polling de `GET /api/documents` cada ~2 s mientras exista algún `pending` (sin websockets ni SSE en MVP, coherente con D-13) | Confirmarlo en F4 al implementar T4.3 |
