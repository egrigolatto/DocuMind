# Notas de aprendizaje — DocuMind

> Última actualización: 2026-10-30
> Propósito: registro vivo del objetivo 2 del proyecto. Cada fase deja acá sus explicaciones, experimentos y conclusiones. No reemplaza a `architecture.md` (qué se construye) ni a `decisions.md` (por qué se decidió): acá va el *cómo funciona por dentro*.

## Cómo se usa este documento

- Al cerrar cada fase, se agrega una entrada con la fecha y la fase: conceptos explicados, experimentos realizados (con resultados reales) y limitaciones descubiertas.
- Las explicaciones son del estilo "tutor técnico": primero el problema, después la mecánica, al final cómo se ve en el código de DocuMind.
- Los experimentos se reproducen con un script o test citado en la entrada, para que "aprendido" signifique "verificado".

## Mapa conceptual del RAG (índice de lo que se aprenderá)

| # | Concepto | Se desarrolla en | Estado |
| --- | --- | --- | --- |
| 1 | Qué es un embedding y por qué "significado ≈ cercanía geométrica" | F3 (antes: intuición en F1 al ver `vector(N)`) | Pendiente |
| 2 | Tokenización y ventana de contexto de un LLM | F2 (chunking) y F6 (armado del prompt) | Pendiente |
| 3 | Estrategias de chunking: tamaño, overlap, por estructura | F2 | Pendiente |
| 4 | Similitud coseno vs L2 vs dot product | F1 (operadores de pgvector) y F5 | Pendiente |
| 5 | Búsqueda aproximada: por qué `ORDER BY embedding <=> q` (distancia coseno) escala y qué hacen HNSW/IVFFlat (y por qué el índice debe coincidir con el operador de la consulta) | F5 | Pendiente |
| 6 | top-k, recall vs precisión, umbrales de "no hay contexto" | F5 | Pendiente |
| 7 | Prompt de generación con grounding: system prompt, formato de contexto, citas | F6 | Pendiente |
| 8 | Por qué RAG reduce alucinaciones (y cuándo no) | F6/F7 | Pendiente |
| 9 | Temperature, tokens de salida, costos por uso | F6 | Pendiente |
| 10 | Inyección de prompts desde contenido recuperado | F7 | Pendiente |
| 11 | Evaluación de un pipeline RAG (manual → batería de pruebas) | F7/F8 | Pendiente |

## N-01 — Estado del arte verificado para DocuMind (Fase de descubrimiento)

**Fecha**: 2026-10-29 · **Fase**: Descubrimiento/planificación

Experimento real: consulta directa a los endpoints públicos de OpenRouter y al registro de npm.

**Hallazgos:**

1. **OpenRouter expone embeddings.** El endpoint `/api/v1/embeddings` existe (responde 401 sin credenciales) y hay un catálogo público en `/api/v1/embeddings/models` con **38 modelos** a la fecha: desde gratuitos (`nvidia/nemotron-3.5-embed-1b:free`, contexto 32.768) hasta pagos (`openai/text-embedding-3-small`, ~$0,02/M tokens; `perplexity/pplx-embed-v1-0.6b`, ~$0,004/M).
2. **El catálogo NO publica las dimensiones de cada modelo.** El campo `dimensions` no aparece en la respuesta. Lección: un modelo de embeddings no es intercambiable sin más — la dimensión fija la columna `vector(N)` de pgvector, y cambiar de modelo implica reindexar todo. Por eso cada documento guardará `embedding_model` y `embedding_dims` (D-10).
3. **El modelo elegido es multimodal.** `nvidia/nemotron-3.5-embed-1b:free` declara en el catálogo entrada `text+image` (pensado también para incrustar páginas como imagen). DocuMind **usa solo su modalidad texto**: nunca envía imágenes. Que el modelo "vea" imágenes no lo convierte en OCR de nuestro pipeline: la extracción de texto sigue siendo local y el OCR por imagen está fuera de alcance (D-10, RNF-06).
4. **OpenRouter normaliza el schema tipo OpenAI** para chat (`/api/v1/chat/completions`), devuelve `usage` con tokens y campo `cost` en créditos — base del registro de costos (D-15/RNF-07).
5. OpenRouter también ofrece **plugins** (`file-parser` para PDFs, `web`, etc.) y una **API de rerank** (hay un cookbook "RAG with Embeddings & Rerank"). Decidido: el parsing de PDF es local (objetivo de aprendizaje); el rerank queda para F8 como mejora medible de recuperación.
6. Versiones a 2026-10-29 en npm: Next 16.4.3, React 19.3.1, TypeScript **7.0.2** (línea muy reciente, ver P-02), drizzle-orm 0.45.4, vitest 5.0.5, zod 4.6.7, unpdf 2.0.1, pdf-parse 2.4.5. Ninguna se da por buena hasta verificarla en F0.

**Limitación de esta nota:** son snapshots de un día; las dependencias y el catálogo de modelos cambian. F0/F1 vuelven a verificar.

## N-02 — Qué versionar, qué ignorar y por qué (Fase 0, T0.1)

**Fecha**: 2026-10-30 · **Fase**: F0 (T0.1)

**Problema**: un repositorio debe contener lo que define al proyecto, no lo que se puede regenerar a partir de eso. Versionar de más infla el historial y produce diffs falsos; versionar de menos rompe la reproducibilidad; versionar secretos los filtra para siempre (quedan en el historial aunque se borren después).

**Qué ignora cada bloque de `.gitignore`:**

1. `node_modules/` — se reconstruye con `pnpm install` a partir de `package.json` + `pnpm-lock.yaml`. Versionarlo sumaría decenas de miles de archivos y diferencias por plataforma sin valor.
2. `.next/`, `out/`, `build/`, `dist/`, `coverage/` — salidas de build y de tests; se regeneran. Si se versionaran, cambiar una config produciría un diff de miles de archivos que no representa trabajo real.
3. `.env*` — contiene secretos (`OPENROUTER_API_KEY`) y configuración local. Un secreto commiteado es un secreto comprometido.
4. `!.env.example` — la excepción que documenta el **contrato de configuración**: nombres de variables y valores de ejemplo, sin secretos. Es documentación ejecutable: `cp .env.example .env` deja la app a un paso de arrancar, y quien clona sabe exactamente qué necesita.
5. `*.log`, `*.tsbuildinfo`, `.DS_Store`, `Thumbs.db`, `.idea/`, `.vscode/` — ruido de la máquina o caché incremental; no aportan al proyecto compartido.

**Regla mental**: se versiona lo que *define* el proyecto (código, docs, configs, lockfile, migraciones); se ignora lo que *se deriva* de eso (instalación, build) o pertenece a la máquina/persona (`.env`, cachés, editor).

**En DocuMind**: `.nvmrc` contiene `24` y sí se versiona porque es parte del contrato del entorno (la máquina local corre v24.13.0); el lockfile de pnpm se suma con la misma lógica en T0.5.

## Entradas por fase

### F0 — Preparación del entorno
**Estado: En curso** — T0.1 completada (ver N-02). Temas previstos para el cierre: contenedores e imágenes (qué es un tag y por qué fijamos uno), por qué un lockfile hace reproducible un proyecto, y el rol de una migración SQL versionada.

### F1 — Esquema de datos y pgvector
**Estado: Pendiente** — temas previstos: el tipo `vector(N)` y qué significa que el espacio vectorial tenga N dimensiones; operadores de distancia en pgvector (`<->` euclídea, `<=>` distancia coseno, `<#>` producto punto **negativo** — ojo con el signo al leer resultados) y cuál usará DocuMind (`<=>`) y por qué la normalización hace que el ranking coseno y el euclídeo coincidan; el experimento del probe de dimensiones (P-01) con su resultado real.

### F2 — Extracción y chunking
**Estado: Pendiente** — temas previstos: por qué un PDF no es texto lineal (layout vs contenido); qué es un token y cómo estimarlo sin tokenizer nativo; trade-off de overlap (contexto roto vs duplicación); resultados del prototipo con PDFs reales (D-11).

### F3 — Embeddings
**Estado: Pendiente** — temas previstos: demo geométrica con dos frases (distancia coseno a mano vs lo que devuelve el modelo); batching y rate limits observados con el modelo `:free` (medición real de tiempos/errores 429).

### F4 — Ingesta end-to-end
**Estado: Pendiente** — temas previstos: multipart en route handlers de Next; estados de un pipeline con pasos que fallan a mitad; límites de timeout reales medidos.

### F5 — Búsqueda semántica
**Estado: Pendiente** — temas previstos: k y umbrales medidos con preguntas reales (P-06); índice vectorial elegido y por qué (P-09); por qué la búsqueda lexical (LIKE) falla donde la semántica funciona, con ejemplos del corpus propio.

### F6 — Generación con citas
**Estado: Pendiente** — temas previstos: anatomía del prompt de RAG (instrucción + contexto numerado + pregunta); cómo se mapea "fuente citada" a IDs de chunk; modelo default elegido con datos (P-04); costo real observado de una respuesta.

### F7 — Robustez
**Estado: Pendiente** — temas previstos: modos de fallo del pipeline observados uno por uno; prompt injection demostrada con un documento malicioso de prueba y la mitigación aplicada.

### F8 — Backlog (solo si la fase se ejecuta)
**Estado: Pendiente / opcional** — F8 no genera nota obligatoria; si alguna tarea de backlog se ejecuta (streaming, rerank, persistencia de chat), su concepto se documenta acá igual que en F0-F7.
