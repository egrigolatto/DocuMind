# Plan de tareas — DocuMind

> Última actualización: 2026-10-30
> Estado general: **En curso** — Fase 0 (T0.1 completada; commit pendiente del owner)

## Definición de terminado (aplica a cada tarea)

Una tarea está terminada cuando:

1. El código funciona como describe su criterio de aceptación (verificado, no asumido).
2. `typecheck`, `lint` y los tests pasan (`pnpm typecheck`, `pnpm lint`, `pnpm test`; antes de que los scripts exista T0.7, equivalente con `pnpm exec`).
3. Si la tarea tocó decisiones de diseño: se actualizó `docs/decisions.md` (nueva entrada, cambio de estado o marca de Superado) y, si aprendimos algo de IA/RAG, se registró una entrada en `docs/learning-notes.md`.
4. Los cambios fueron revisados por el owner y commiteados por él bajo Conventional Commits (la rama y el mensaje se proponen, nunca se ejecutan automáticamente).
5. Si la implementación se desvió del plan, este `TASKS.md` quedó actualizado.

**Regla de trabajo:** se ejecuta una tarea a la vez, con propuesta previa y aprobación del owner. Las fases F2–F8 están como hoja de ruta; se desglosan al llegar a ellas. **Al desglosar, cada unidad verificable recibe su propio criterio de aceptación** (no se aceptan tareas multi-unidad del tipo "validación + orquestación + estados" en una sola tarea).

## Fase 0 — Preparación del entorno

- [x] T0.1 — Inicializar repositorio Git
  - Objetivo: base versionada del proyecto con estructura de ramas acordada (D-16).
  - Componentes: `.git`, `.gitignore` (incluye `.env`, `.next/`, `node_modules/`, volumen Docker), `.nvmrc`.
  - Dependencias: ninguna.
  - Criterio de aceptación: `git status` muestra working tree limpio tras el commit inicial; `.gitignore` excluye `.env`; `node -v` coincide con `.nvmrc` (24.x).
  - Nota de aprendizaje: qué fija cada archivo ignorado y por qué `.env.example` sí se versiona.
  - **Completada 2026-10-30** — verificado: `.env` ignorado (exit 0), `.env.example` versionable (exit 1), Node v24.13.0 con `.nvmrc` = 24. Commit inicial pendiente del owner.

- [ ] T0.2 — Scaffold de Next.js y fijación de versiones
  - Objetivo: aplicación Next 16 con App Router, TypeScript, Tailwind y ESLint, usando pnpm; resolver P-02.
  - Componentes: `src/app/`, `tsconfig.json`, `next.config.ts`, configs de lint/tailwind.
  - Dependencias: T0.1.
  - Criterio de aceptación: `pnpm dev` sirve la página en `localhost:3000`; `pnpm build` pasa; versiones finales de Next/React/TypeScript anotadas en `docs/decisions.md` (cierre de P-02 y registro si hubo fallback de TS).

- [ ] T0.3 — PostgreSQL + pgvector en Docker Compose
  - Objetivo: base de datos reproducible local (RNF-03); resolver P-03.
  - Componentes: `docker-compose.yml`, volumen persistente, healthcheck, credenciales locales.
  - Dependencias: T0.1.
  - Criterio de aceptación: `docker compose up -d` deja Postgres saludable en el puerto configurado; `CREATE EXTENSION vector` funciona en el contenedor; tag exacto de la imagen registrado en `docs/decisions.md` (cierre de P-03).

- [ ] T0.4 — Variables de entorno y validación con Zod
  - Objetivo: configuración tipada y fail-fast (RNF-01).
  - Componentes: `.env.example` (`DATABASE_URL`, `OPENROUTER_API_KEY`, `OPENROUTER_EMBEDDINGS_MODEL`, `OPENROUTER_CHAT_MODEL`), `src/lib/env.ts` (aún vacío de lógica de app: solo schema y parseo).
  - Dependencias: T0.2.
  - Criterio de aceptación: con `.env` ausente o incompleto, la app falla al arrancar con mensaje claro indicando la variable faltante; `src/lib/env.ts` exportado da tipos exactos; test unitario cubre caso válido e inválido.
  - Nota: `OPENROUTER_CHAT_MODEL` puede aceptarse vacío/placeholder hasta F6 (donde se decide su valor, P-04); `OPENROUTER_EMBEDDINGS_MODEL` tiene default documentado (`nvidia/nemotron-3.5-embed-1b:free`).

- [ ] T0.5 — Instalación y fijación de dependencias core
  - Objetivo: agregar drizzle-orm, pg, zod (prod) y drizzle-kit, @types/pg, vitest, vite (dev — Vitest 5 exige `vite ^6 || ^7 || ^8` como peer, verificado en revisión), verificando la versión actual de cada una antes de instalar (no se asumen las de architecture.md).
  - Componentes: `package.json`, `pnpm-lock.yaml`, `drizzle.config.ts` (skeleton apuntando a `src/modules/db/schema.ts`, que se crea en F1).
  - Dependencias: T0.2, T0.3.
  - Criterio de aceptación: `pnpm install` resuelve sin errores; las versiones instaladas coinciden con las verificadas ese día; cualquier diferencia relevante con `docs/architecture.md` se actualiza ahí.

- [ ] T0.6 — Cliente de base de datos y smoke test de integración
  - Objetivo: primera conexión real Node ↔ Postgres del proyecto; Vitest configurado.
  - Componentes: `src/modules/db/client.ts`, `vitest.config.ts`, test de integración (`SELECT 1` contra el contenedor).
  - Dependencias: T0.4, T0.5.
  - Criterio de aceptación: `pnpm exec vitest run` pasa la unidad (el script `pnpm test` recién existe desde T0.7); el test de integración pasa con `docker compose up` y se salta con mensaje claro sin él; `pnpm exec tsc --noEmit` verde.

- [ ] T0.7 — Scripts de proyecto
  - Objetivo: flujo de trabajo estándar documentado en `package.json`.
  - Componentes: scripts `dev`, `build`, `start`, `lint`, `typecheck`, `test`, `test:integration`, `db:generate`, `db:migrate`, `db:studio`.
  - Dependencias: T0.5, T0.6.
  - Criterio de aceptación: todos los scripts corren sin error en el estado actual; el README (sección Instalación) actualizado con los comandos ya reales, quitando el aviso "pendiente de ejecutar".

**✔ Fin de Fase 0:** cuando T0.1–T0.7 estén completas, F0 se marca terminada y se revisa este plan antes de desglosar/ajustar F1.

## Fase 1 — Esquema de datos y pgvector

- [ ] T1.1 — Probe de dimensiones del modelo de embeddings (cierra P-01)
  - Objetivo: obtener la dimensión real de `nvidia/nemotron-3.5-embed-1b:free` antes de fijar `vector(N)`; primer contacto verificado con la API de embeddings de OpenRouter (base de D-10).
  - Componentes: script desechable o test manual (sin crear todavía módulo `ai/` completo), usando `OPENROUTER_API_KEY`.
  - Dependencias: T0.4 (requiere `.env` con key real del owner).
  - Criterio de aceptación: la respuesta del endpoint es válida; dimensión documentada en `docs/decisions.md` (cierre de P-01) y explicada en `docs/learning-notes.md` (entrada F1: qué devolvió, forma del vector, primera intuición de "significado como geometría"); respuesta validada campo por campo con Zod en el propio script.

- [ ] T1.2 — Esquema Drizzle: `documents` y `chunks`
  - Objetivo: modelo de datos del diseño (D-07) con columna vector de la dimensión de T1.1.
  - Componentes: `src/modules/db/schema.ts`.
  - Dependencias: T1.1.
  - Criterio de aceptación: columnas y tipos según `docs/architecture.md` §Modelo de datos (incluye `status`, `embedding_model`, `embedding_dims`, FK con cascade); PKs como **uuid** tras confirmar P-10; `pnpm db:generate` produce SQL legible que crea ambas tablas y la extensión.

- [ ] T1.3 — Primera migración aplicada
  - Objetivo: historial de migraciones Drizzle funcionando contra el contenedor (RNF-04).
  - Componentes: `src/modules/db/migrations/`, script `db:migrate`.
  - Dependencias: T1.2, T0.3.
  - Criterio de aceptación: migración aplica limpia en base vacía; `db:studio` muestra el esquema; re-ejecutar es no-op sin error.

- [ ] T1.4 — Repositorios de documentos y chunks
  - Objetivo: acceso a datos tipado y testeado, único camino del dominio a la base.
  - Componentes: `src/modules/db/repositories/documents.ts`, `src/modules/db/repositories/chunks.ts`.
  - Dependencias: T1.3.
  - Criterio de aceptación: operaciones mínimas (insert document, update status/error, insert chunks en lote, delete cascade, list) con tests de integración que usan un vector real de muestra; borrar un documento elimina sus chunks (verificado); transacción en la inserción por lotes (si falla un chunk, no queda mitad indexado).

- [ ] T1.5 — Nota de aprendizaje pgvector (HU-07)
  - Objetivo: cerrar el objetivo didáctico de F1 antes de pasar a F2.
  - Componentes: entrada F1 en `docs/learning-notes.md`.
  - Dependencias: T1.1–T1.4.
  - Criterio de aceptación: la nota explica con ejemplo propio: qué es `vector(N)`, los tres operadores de distancia (`<->` euclídea, `<=>` coseno, `<#>` producto punto negativo) con resultado numérico real de una consulta en la base local, y por qué DocuMind ordenará por `<=>` (distancia coseno); la relación operador↔índice vectorial se retoma en F5 (P-09).

**✔ Fin de Fase 1.**

## Fase 2 — Extracción de texto y chunking (hoja de ruta; se desglosa al iniciar)

- [ ] T2.1 — Pipeline TXT: lectura + normalización + chunking puro (función testeable) con tamaño y overlap configurables (RF-02, RF-03; resuelve P-05 con experimentos).
- [ ] T2.2 — Extracción PDF con unpdf sobre 2-3 PDFs reales de prueba; documentar el resultado de la validación de D-11 y reabrir decisión si la calidad es insuficiente (RF-02).
- [ ] T2.3 — Casos límite: PDF sin texto seleccionable y PDF con contraseña (rechazo con mensaje claro; sin OCR ni descifrado), archivos Unicode/acentos, documentos vacíos, subida duplicada (nuevo documento, sin dedupe) (RF-01, RF-09; criterios de HU-01).
- [ ] T2.4 — Nota de aprendizaje F2 en learning-notes (tokens, chunking, layout de PDF) (HU-07).

## Fase 3 — Embeddings (hoja de ruta)

- [ ] T3.1 — Cliente OpenRouter en `src/modules/ai/`: fetch + tipos Zod + reintentos con backoff (D-06, D-10, RNF-08).
- [ ] T3.2 — Generación de embeddings por lotes de chunks + persistencia en `chunks.embedding` (RF-04).
- [ ] T3.3 — Manejo de rate limit del free-tier observado en la práctica; ajustes documentados (riesgo R-02; seguimiento P-01).
- [ ] T3.4 — Nota de aprendizaje F3 (demo geométrica de similitud con embeddings reales) (HU-07).

## Fase 4 — Ingesta end-to-end (hoja de ruta)

- [ ] T4.1 — `POST /api/documents`: validación multipart + Zod (tipo real, tamaño ≤5 MB provisional según RF-01), orquestación extract→chunk→embed→`ready`/`error`; criterio obligatorio: si algo falla en cualquier paso, el documento queda en `error` con motivo y sin chunks huérfanos (transacción única de inserción; regla de D-09, criterio de HU-06) (RF-01, RF-09; resuelve P-08).
- [ ] T4.2 — `GET /api/documents` y `DELETE /api/documents/[id]` (RF-05, RF-06).
- [ ] T4.3 — UI: formulario de subida + listado con estados, con polling para ver `pending`→`ready` sin intervención (mecanismo P-12) (HU-01, HU-02; RF-05).
- [ ] T4.4 — Nota de aprendizaje F4 + actualización de límites reales en README (HU-07).

## Fase 5 — Búsqueda semántica (hoja de ruta)

- [ ] T5.1 — Embedding de la pregunta + búsqueda top-k por similitud coseno en `retrieve.ts` (RF-07 parcial; resuelve P-06 y P-09 con mediciones).
- [ ] T5.2 — Umbral de "sin contexto relevante" (base del criterio de HU-03 sobre preguntas sin relación).
- [ ] T5.3 — Batería de preguntas de prueba contra documentos reales; resultados en learning-notes F5.

## Fase 6 — Respuesta con fuentes + chat (hoja de ruta)

- [ ] T6.1 — Prompt de RAG con contexto numerado e instrucción de citas; modelo default decidido con datos (resuelve P-04; RF-07).
- [ ] T6.2 — `POST /api/chat`: retrieve → generate → respuesta + fuentes tipadas (RF-07, RF-08).
- [ ] T6.3 — UI de chat con historial en memoria de sesión (RF-10; HU-03, HU-04).
- [ ] T6.4 — Registro de `usage`/costo por respuesta (D-15, RNF-07).
- [ ] T6.5 — Nota de aprendizaje F6 (anatomía del prompt, grounding, citas, costo real observado) (HU-07).

## Fase 7 — Robustez y cierre (hoja de ruta)

- [ ] T7.1 — Pasada de errores end-to-end: cada fallo externo tiene mensaje diferenciado en UI (RF-09, HU-06).
- [ ] T7.2 — Medición de latencia y costos reales vs RNF-07/RNF-10; ajuste de límites (confirma o corrige RNF-10 Propuesto).
- [ ] T7.3 — Demo de prompt injection con documento malicioso de prueba + mitigación aplicada (D-17, riesgo R-08).
- [ ] T7.4 — Documentación final consistente con lo construido: comandos del README y rutas de architecture verificadas uno a uno contra el repo (HU-07).
- [ ] T7.5 — Datos de demo empaquetados (PDFs/TXT de ejemplo + script de carga en `scripts/`).

## Fase 8 — Backlog (no comprometido)

- [ ] T8.1 — Streaming SSE de respuestas (D-13, reevaluar).
- [ ] T8.2 — Persistencia de conversaciones y multiusuario (reevaluación de D-08/D-12).
- [ ] T8.3 — Rerank con la API de OpenRouter como mejora medible de retrieval.
- [ ] T8.4 — Reprocesamiento/re-chunking de documentos existentes: exige primero persistir el binario original o el texto fuente (decisión pendiente P-11; hoy re-procesar = re-subir, según D-07).
- [ ] T8.5 — CI (lint + typecheck + tests).

## Registro de desviaciones

| Fecha | Tarea | Desviación | Decisión registrada |
| --- | --- | --- | --- |
| — | — | — | — |
