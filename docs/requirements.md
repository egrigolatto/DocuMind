# Requisitos — DocuMind

> Última actualización: 2026-10-29
> Estado: Aprobado

## Historias de usuario

| ID | Historia | Prioridad |
| --- | --- | --- |
| HU-01 | Como usuario, quiero cargar archivos PDF o TXT para que DocuMind pueda indexar su contenido. | Alta |
| HU-02 | Como usuario, quiero ver los documentos cargados y su estado de procesamiento para saber si ya son consultables. | Alta |
| HU-03 | Como usuario, quiero hacer preguntas en un chat para obtener información contenida en mis documentos. | Alta |
| HU-04 | Como usuario, quiero ver las fuentes de cada respuesta para verificar que no está inventada. | Alta |
| HU-05 | Como usuario, quiero eliminar documentos para limpiar la base de datos entre demos. | Media |
| HU-06 | Como usuario, quiero que el sistema me informe claramente cuando algo falla (archivo inválido, API de IA caída) para no quedarme sin respuestas. | Alta |
| HU-07 | Como desarrollador/portfolio, quiero entender cada etapa del pipeline RAG para poder explicarla en una entrevista técnica. | Alta |

## Requisitos funcionales

| ID | Requisito | Origen | Estado |
| --- | --- | --- | --- |
| RF-01 | El sistema debe permitir cargar documentos PDF (con texto seleccionable) y archivos TXT, validando tipo y tamaño máximo. Valor inicial **5 MB — Propuesto** y provisional; se ajusta con la medición de P-08 en F4. | HU-01 | Confirmado (validación) / Propuesto (el valor 5 MB) |
| RF-02 | El sistema debe extraer el texto de los documentos cargados (PDF con unpdf; TXT lectura directa). | HU-01 | Confirmado |
| RF-03 | El sistema debe dividir el texto extraído en *chunks* con tamaño y overlap configurables. | HU-01 / HU-07 | Confirmado |
| RF-04 | El sistema debe generar un embedding por chunk vía OpenRouter y persistirlo junto al texto y su documento de origen. | HU-01 | Confirmado |
| RF-05 | El sistema debe listar los documentos con su estado de procesamiento (`pending` / `ready` / `error`). | HU-02 | Confirmado |
| RF-06 | El sistema debe permitir eliminar un documento cascando sus chunks y vectores. | HU-05 | Confirmado |
| RF-07 | El sistema debe responder preguntas del usuario recuperando los *k* chunks más relevantes por similitud semántica y enviándolos como contexto a un LLM vía OpenRouter. | HU-03 | Confirmado |
| RF-08 | Cada respuesta debe identificar las fuentes utilizadas (documento y fragmento). | HU-04 | Confirmado |
| RF-09 | El sistema debe validar entradas en todos los bordes (Zod) y devolver errores tipados y visibles en la UI. | HU-06 | Confirmado |
| RF-10 | El chat debe mantener el historial de la conversación durante la sesión del navegador; no se persiste en la base. | HU-03 | Confirmado |

## Requisitos no funcionales

| ID | Requisito | Categoría | Estado |
| --- | --- | --- | --- |
| RNF-01 | TypeScript en modo estricto en todo el proyecto; Zod obligatorio en env, entrada HTTP y respuestas de APIs externas. | Mantenibilidad | Confirmado |
| RNF-02 | Pruebas con Vitest donde aportan: lógica pura (chunking, normalización, mapeos) e integración (repositorios, búsqueda vectorial). No se escribe cobertura vacía. | Calidad | Confirmado |
| RNF-03 | Entorno reproducible: `docker compose up` levanta la base; `.env.example` sin secretos reales. | Portabilidad | Confirmado |
| RNF-04 | Cambios de esquema de base de datos exclusivamente mediante migraciones Drizzle versionadas en el repo. | Mantenibilidad | Confirmado |
| RNF-05 | Sin autenticación ni multiusuario en el MVP; la app es un solo tenant local: todos los visitantes de una instancia comparten el índice de documentos (D-08). | Alcance | Confirmado (exclusión explícita) |
| RNF-06 | Sin OCR, sin servidores MCP, sin agentes autónomos, sin procesamiento distribuido, colas ni microservicios. | Alcance | Confirmado (exclusión explícita) |
| RNF-07 | Costos de IA controlados: límite de tamaño por documento y registro del `usage`/costo que devuelve OpenRouter en cada llamada. | Costo | Confirmado |
| RNF-08 | Las claves de API se usan solo del lado del server; el cliente nunca recibe `OPENROUTER_API_KEY`. Las llamadas a OpenRouter ocurren exclusivamente en código de servidor. | Seguridad | Confirmado |
| RNF-09 | Documentación en español; código, identificadores y convenciones (commits, ramas) en inglés. | Documentación | Confirmado |
| RNF-10 | Latencia objetivo: respuesta del chat en menos de ~20 s en condiciones normales. **Propuesto** como valor de referencia del equipo (no exigencia del usuario); se confirma o corrige midiendo en F7 (T7.2). | Rendimiento | Propuesto |
| RNF-11 | Cada módulo del pipeline debe ser legible y explicable sin depender de frameworks RAG (no LangChain ni equivalentes). El flujo fundamental se implementa con TypeScript y llamadas HTTP directas. | Aprendizaje | Confirmado |
| RNF-12 | Commits bajo Conventional Commits; desarrollo en ramas por tarea/fase; sin commits ni operaciones destructivas automáticos. | Proceso | Confirmado |

## Criterios de aceptación

### HU-01 / RF-01 — Carga de documentos
- Dado un PDF con texto seleccionable de ≤5 MB (límite provisional), cuando se sube desde el panel, entonces el documento aparece en el listado con estado `pending` y luego `ready`.
- Dado un archivo `.exe` renombrado `.txt`, cuando se sube, entonces el sistema lo rechaza con un mensaje de error claro (validación de tipo real, no solo extensión).
- Dado un PDF escaneado (sin texto seleccionable), cuando se sube, entonces el sistema lo marca con estado `error` y un mensaje informativo claro (OCR está fuera de alcance).
- Dado un PDF protegido con contraseña, cuando se sube, entonces el sistema lo rechaza con mensaje claro (descifrado fuera de alcance — **Propuesto**, T2.3).
- Dado un documento ya cargado previamente, cuando se sube otra vez, entonces se crea como documento nuevo independiente: **no hay detección de duplicados en el MVP** (eliminado duplica solo su copia).

### HU-02 / RF-05 — Listado y estados
- Dado un documento en procesamiento, cuando el usuario tiene el listado visible, entonces su estado pasa de `pending` a `ready`/`error` sin intervención manual (mecanismo: polling — Propuesto, P-12).
- Dado un documento cuyo procesamiento falló, entonces el listado muestra `error` con un motivo legible.

### HU-03, HU-04 / RF-07, RF-08 — Chat con fuentes
- Dado al menos un documento `ready`, cuando el usuario pregunta algo respondible con su contenido, entonces la respuesta se basa en ese contenido y cita documento y fragmento.
- Dada una pregunta sin relación con los documentos, cuando se consulta, entonces el sistema indica que no encontró información relevante en lugar de inventar (umbral de similitud y diseño de prompt de F5/F6).
- Dado el refresh del navegador, cuando el usuario vuelve al chat, entonces el historial se pierde (comportamiento esperado por RF-10).

### HU-05 / RF-06 — Eliminación
- Dado un documento `ready`, cuando se elimina, entonces desaparecen su registro, sus chunks y sus vectores en cascada, y las respuestas posteriores no lo citan.

### HU-06 / RF-09 — Errores
- Dado que OpenRouter responde 401/429/5xx, cuando se procesa una solicitud, entonces el usuario ve un mensaje de error diferenciado (configuración / rate limit / servicio).
- Dado que la ingesta falla en cualquier paso (extracción, embedding, corte de conexión), entonces el documento termina en `error` con `error_message` y **no quedan chunks huérfanos**: la inserción de chunks+vectores es una única transacción (regla de D-09; se verifica en T4.1).

### HU-07 — Comprensión del pipeline (objetivo de aprendizaje)
- Dado el cierre de cada fase, cuando se revisa `docs/learning-notes.md`, entonces existe una entrada con el concepto explicado, el experimento que lo verificó y su resultado real.
- Dado el cierre de F6, entonces las siete etapas del pipeline (extracción → chunking → embedding → almacenamiento → retrieval → prompt → generación con citas) están explicadas en learning-notes sin términos que el owner no pueda definir por sí mismo.

## Fuera de alcance (MVP)

- **Autenticación y gestión de usuarios:** decisión explícita del owner; se evalúa en futuras versiones. Consecuencia: todos los visitantes comparten el mismo índice de documentos.
- **OCR de documentos escaneados:** la extracción cubre solo PDF con texto seleccionable y TXT.
- **Servidores MCP y agentes autónomos:** fuera del objetivo de aprendizaje del flujo fundamental.
- **Procesamiento distribuido, colas o microservicios:** la ingesta es síncrona por request (ver D-09 en decisions.md).
- **Persistencia del historial de chat:** el historial vive en el navegador (decisión D-12).
- **Streaming de respuestas (SSE):** se considera en F8 (decisión D-13).
- **Integraciones externas adicionales:** solo OpenRouter.
