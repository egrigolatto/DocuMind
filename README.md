# DocuMind

Asistente inteligente de documentos basado en RAG (Retrieval-Augmented Generation): cargás PDF y TXT, y consultás su contenido mediante un chat que responde con base en tus documentos y cita las fuentes utilizadas.

## Objetivo

DocuMind tiene dos objetivos inseparables:

1. **Producto de portfolio:** una aplicación web funcional y presentable que demuestra un pipeline RAG completo (extracción → chunking → embeddings → búsqueda vectorial → generación con citas).
2. **Proyecto de aprendizaje:** comprender cómo funciona cada etapa —modelos de lenguaje, embeddings, búsqueda semántica con pgvector— sin abstracciones que oculten el flujo. Cada decisión técnica está documentada y justificada.

## Características (MVP)

- Carga de PDF (con texto seleccionable) y TXT, con validación de tipo y tamaño.
- Pipeline de indexación propio: extracción, *chunking*, embeddings y búsqueda semántica con pgvector.
- Chat con respuestas fundamentadas y fuentes citadas.
- Gestión simple de documentos (listado con estado, eliminación).
- Entorno reproducible con Docker y validaciones estrictas (Zod, TypeScript).

El detalle de requisitos está en [docs/requirements.md](docs/requirements.md).

## Tecnologías

Next.js (App Router) · React · TypeScript · Tailwind CSS · pnpm · PostgreSQL + pgvector · Drizzle ORM · Zod · Vitest · Docker Compose · OpenRouter (embeddings y chat, vía `fetch` directo, sin frameworks RAG).

Cada tecnología con su justificación y alternativas evaluadas: [docs/architecture.md](docs/architecture.md#stack-tecnológico) y [docs/decisions.md](docs/decisions.md).

## Alcance del MVP

**Single-tenant:** no hay autenticación ni cuentas. Todos los visitantes de una instancia comparten el mismo índice de documentos (D-08). Fuera de alcance: OCR de escaneados, streaming de respuestas, persistencia del historial de chat. Ver [fuera de alcance](docs/requirements.md#fuera-de-alcance-mvp).

## Requisitos previos

- Node.js 24.x (fijado en `.nvmrc` durante la Fase 0).
- pnpm 11.15.1 (fijada en `packageManager`).
- Docker con Docker Compose.
- Una cuenta y API key de [OpenRouter](https://openrouter.ai) (los embeddings elegidos son de un modelo gratuito; el modelo de chat se define con datos en la Fase 6).

## Instalación

> ⚠️ **Fase 0 en curso:** `pnpm install`, `pnpm dev` y `docker compose up -d` ya funcionan (T0.2–T0.3); `.env.example` y `pnpm db:migrate` se habilitan en T0.4–T0.5. Esta sección se actualiza al cerrar la fase (T0.7).

```bash
pnpm install
cp .env.example .env        # completar las variables de entorno (OPENROUTER_API_KEY, etc.)
docker compose up -d        # PostgreSQL + pgvector
pnpm db:migrate             # aplicar migraciones Drizzle
pnpm dev                    # http://localhost:3000
```

## Uso

Flujo previsto del MVP (disponible al completar las Fases 4 y 6):

1. Abrir la app, subir un PDF o TXT desde el panel de documentos.
2. Esperar a que el documento pase a estado *listo*.
3. Preguntar en el chat algo sobre el contenido.
4. Leer la respuesta y sus fuentes citadas.

## Estructura del proyecto

Entorno base en marcha (T0.2–T0.3): scaffold de Next 16 (`src/app/`, `public/` y configuraciones) y PostgreSQL + pgvector en `docker-compose.yml`. Los módulos de dominio (`src/modules/`, `src/components/`, `src/lib/`) se materializan en el resto de la Fase 0 y en F1. Árbol completo aprobado en [docs/architecture.md](docs/architecture.md#estructura-de-carpetas).

## Documentación

- [Requisitos](docs/requirements.md) — requisitos funcionales, no funcionales, historias de usuario y criterios de aceptación.
- [Arquitectura](docs/architecture.md) — stack, componentes, modelo de datos, integraciones y riesgos.
- [Decisiones](docs/decisions.md) — registro de decisiones técnicas con estado y alternativas.
- [Notas de aprendizaje](docs/learning-notes.md) — conceptos de IA/RAG aprendidos, explicaciones y experimentos por fase.
- [Plan de tareas](TASKS.md) — hoja de ruta y progreso.

## Roadmap

F0 entorno → F1 datos/pgvector → F2 extracción/chunking → F3 embeddings → F4 ingesta → F5 búsqueda semántica → F6 chat con fuentes → F7 robustez → F8 backlog (streaming, historial, rerank, CI). El detalle vive en [TASKS.md](TASKS.md).

## Licencia

**Pendiente** de definir (ver P-07 en docs/decisions.md).
