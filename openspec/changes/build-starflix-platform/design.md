# Design

## Context

El repositorio solo contiene documentación (`docs/`, `Enunciado.md`) y el andamiaje de OpenSpec; no hay código ni infraestructura. Ver `proposal.md` para la motivación.

Restricciones que condicionan el diseño:

- Un único `docker compose up --build` debe dejar el sistema operativo, sin pasos manuales.
- MongoDB y RabbitMQ deben usar imágenes oficiales y vivir dentro del compose.
- El contenedor del frontend no puede resolver ni alcanzar MongoDB.
- Las recomendaciones deben responder en menos de 100 ms y no pueden entrenar durante una petición.
- La curación debe ser idempotente para distinguir arranque en frío de arranque en caliente.
- El LLM es obligatorio para la inferencia de ratings y deseable para traducción y chat.
- El dataset exacto todavía no está en el repositorio; su esquema se inspeccionará al integrarlo.

## Goals / Non-Goals

**Goals:**

- Definir una topología de servicios, redes y volúmenes reproducible en Docker Compose.
- Fijar contratos REST y eventos versionados entre frontend, gateway y microservicios.
- Elegir un stack uniforme que cubra web, datos y machine learning sin multiplicar lenguajes.
- Definir el modelo de datos de MongoDB, incluidos auditoría y estado del sistema.
- Diseñar la curación idempotente, la inferencia con LLM y su trazabilidad.
- Diseñar recomendaciones precalculadas con latencia O(1) por consulta.
- Ordenar la implementación en fases verificables.

**Non-Goals:**

- Autenticación, autorización y registro dinámico de usuarios (fuera del enunciado; ver `docs/notas-y-propuestas.md`).
- Alta disponibilidad, réplicas, orquestación Kubernetes o escalado horizontal.
- Recomendaciones en tiempo real por streaming; la actualización puede ser en segundo plano.
- Entrenamiento de modelos propios de deep learning o serving con GPU.
- Administración manual de RabbitMQ o MongoDB desde el host.

## Decisions

### D1. Stack: backend Python + frontend React/TypeScript

Todos los microservicios backend usan **Python 3.12 + FastAPI + Pydantic**. El frontend usa **React 19 + Vite + TypeScript + React Router + TanStack Query + Tailwind CSS**.

- **Por qué**: un solo lenguaje backend evita inconsistencias estilísticas, y Python concentra el ecosistema necesario para curación (`pandas`), recomendación (`scikit-surprise`, `scikit-learn`) y clientes de LLM. FastAPI aporta validación, async y OpenAPI automático.
- **Alternativas**: Node/TypeScript en todo el backend (peor ecosistema ML); stack políglota Node + Python (más imágenes, más contratos y más superficie de error).

### D2. Descomposición en servicios

| Servicio | Responsabilidad | Persistencia |
|---|---|---|
| `frontend` | SPA servida por Nginx; única publicación al host | Ninguna |
| `api-gateway` | Entrada única, ruteo, validación de borde, caché de estado | Lee `system_state` |
| `catalog-service` | Catálogo, búsqueda, detalle, reseñas, usuarios | Lee `movies`, `reviews`, `users` |
| `interaction-service` | Valida y publica interacciones | Ninguna escritura |
| `interaction-worker` | Consume la cola y persiste reseñas | Escribe `reviews` |
| `recommendation-service` | Precalcula y sirve recomendaciones | Lee `movies`, `reviews`; volumen propio |
| `cleaning-service` | Sanea el dataset, infiere ratings, audita, publica estado | Escribe `movies`, `reviews`, `audit_logs`, `system_state` |
| `ai-service` | Traducción y chat con LLM | Lee catálogo; escribe `translations` |

Cada servicio tiene su propio módulo `repository` para aislar consultas. Solo los servicios que lo necesitan reciben credenciales de escritura.

### D3. Redes Docker y mínimo privilegio

Tres redes internas:

- `frontend_net`: solo `frontend` y `api-gateway`.
- `backend_net`: `api-gateway`, microservicios y `rabbitmq`.
- `data_net`: microservicios con acceso a datos, `mongodb` y `rabbitmq`.

`mongodb` se conecta únicamente a `data_net`; `frontend` únicamente a `frontend_net`. Por lo tanto, el frontend no tiene ruta ni DNS hacia MongoDB. Solo el puerto del frontend se publica al host; MongoDB y RabbitMQ no exponen puertos.

- **Alternativa**: una sola red y reglas de firewall internas. Se descarta porque no garantiza el requisito de no resolución de nombre.

### D4. Contratos REST y eventos

REST JSON a través del gateway, prefijo `/api`:

- `GET /api/movies?page&limit&q&genre`
- `GET /api/movies/{id}`
- `GET /api/movies/{id}/reviews?page&limit`
- `GET /api/users?q`
- `GET /api/recommendations?userId`
- `GET /api/movies/{id}/similar`
- `POST /api/interactions`
- `GET /api/system/status`
- `GET /api/audit/summary`
- `GET /api/audit/entries?type&page&limit`
- `POST /api/ai/translate`
- `POST /api/ai/chat`

Errores uniformes: `{"error": {"code": "...", "message": "..."}}` con códigos HTTP 400, 404, 409, 423, 503. Las operaciones internas usan endpoints `/internal/*` no publicados al host.

RabbitMQ con exchange `starflix.events` (topic, durable) y colas durables:

- `interactions` ← routing key `interaction.created` (publica `interaction-service`, consume `interaction-worker`).
- `interactions.dlq` ← mensajes agotados.
- `catalog.cleaned` ← evento de fin de curación (publica `cleaning-service`, consume `recommendation-service`).
- `recommendations.ready` ← evento de recomendador operativo (publica `recommendation-service`, consume `api-gateway`).
- `state.changed` ← fanout de invalidación de estado (publican `cleaning-service` y `recommendation-service`, consume `api-gateway`).
- `recommendation.refresh` ← nuevas interacciones para actualización incremental.

Los mensajes son persistentes (`delivery_mode=2`), con `messageId` para idempotencia y un sobre `{version: 1, type, occurredAt, payload}`.

### D5. Modelo de datos MongoDB

- `movies`: `_id`, `title`, `year` (int), `runtime` (int), `genres` (list), `directors` (list), `cast` (list), `plot`, `fullplot`, `poster`, `imdb {rating, votes}`, `released` (date), `dataQuality {status, issues}`, `curationVersion`.
- `reviews`: `_id`, `movie_id`, `user_id`, `rating` (int 1–5 o null), `comment`, `date` (date), `source` (`dataset`/`user`), `status` (`valid`/`spam`), `llmInference {applied, model, confidence, justification}`, `curationVersion`.
- `users`: `_id` = `usr_0001`…`usr_2000`, `displayName`.
- `audit_logs`: `runId`, `anomalyType`, `collection`, `documentId`, `field`, `before`, `after`, `reason`, `llm {model, justification, confidence}`, `status`, `createdAt`.
- `system_state`: `_id: "global"`, `curation {status, startedAt, finishedAt, metrics}`, `recommendations {status, indexedAt, version}`, `dataVersion`, `updatedAt`.
- `translations`: `_id` = hash(texto + idioma + modelo), `sourceHash`, `language`, `translated`, `model`, `createdAt`.
- `movies_quarantine`: documentos duplicados o irrecuperables, para auditoría.

Índices mínimos: `movies.title` (texto o normalizado), `movies.genres`, `reviews.movie_id`, `reviews.user_id`, único `(user_id, movie_id)`, `audit_logs.anomalyType`, `audit_logs.runId`.

### D6. Curación idempotente

El `cleaning-service` ejecuta un pipeline por fases al arrancar:

1. **Preliminar**: adquiere un lock en `system_state`/colección de locks con heartbeat y detecta si `curation.status == "done"` y `dataVersion` coincide.
2. **Películas**: deduplicación por `(título normalizado, año)`, normalización de año, duración, géneros, rating y título.
3. **Reseñas**: normalización de fechas, detección de spam, eliminación de huérfanas y consolidación de votos duplicados.
4. **Inferencia LLM**: lotes de reseñas con `rating: null` y comentario evaluable.
5. **Cierre**: métricas consolidadas en `system_state`, estado `done`, publicación de `catalog.cleaned`.

Cada corrección escribe un `audit_logs` con antes/después y motivo. Los documentos llevan `curationVersion` para no reprocesarse. Los duplicados se mueven a `movies_quarantine` en lugar de borrarse, para conservar evidencia. Si el proceso se interrumpe, el lock expira y la siguiente ejecución retoma lo pendiente.

### D7. LLM: abstracción de proveedor con Ollama por defecto

`cleaning-service` y `ai-service` usan una interfaz común `LLMProvider` con dos implementaciones:

- `ollama`: contenedor oficial `ollama/ollama` con un modelo pequeño (`qwen2.5:1.5b-instruct`), descargado automáticamente al primer arranque y persistido en un volumen.
- `openai-compatible`: endpoint y credenciales por variables de entorno, para usar una API externa.

La inferencia de rating pide JSON estricto `{rating, confidence, justification}`, valida rango entero 1–5 y reintenta ante respuestas inválidas. Se usa temperatura baja, concurrencia limitada (4–8), caché por hash de comentario y timeout con degradación a “pendiente”.

- **Por qué Ollama por defecto**: permite que la evaluación funcione sin claves externas y mantiene el requisito de un único comando. La primera ejecución requiere internet para descargar el modelo; el volumen lo conserva.
- **Alternativas**: solo API externa (requiere clave manual); clasificador de sentimiento embebido (no cumple el espíritu de “modelo de lenguaje” y explica peor la decisión).

### D8. Recomendaciones híbridas precalculadas

En la indexación (arranque o evento `catalog.cleaned`):

- **Contenido**: `metadata soup` con géneros, director, reparto y sinopsis; vectorización TF-IDF; similitud coseno; top-20 de similares por película.
- **Colaborativo**: SVD o KNN con `scikit-surprise` sobre `(user_id, movie_id, rating)`.
- **Baseline**: rating ponderado tipo IMDb para usuarios sin historial.
- **Dos etapas**: candidatos por SVD + vecinos de contenido de las películas mejor valoradas; re-ranking por score combinado y penalización de ya vistos.

Se precalcula el top-50 personalizado por usuario y el top-20 de similares por película. Los artefactos se guardan en el volumen `recommendation_data` (formato Parquet/JSON) y se cargan en memoria. La consulta es una búsqueda directa por `user_id` o `movie_id`, sin cálculo pesado.

La actualización por nuevas interacciones es incremental: se refrescan en segundo plano los usuarios afectados y la película calificada. Si la versión de datos cambia, se reconstruye todo en segundo plano manteniendo la versión anterior disponible.

- **Alternativa**: calcular similitud o factores por request. Se descarta por riesgo de latencia y por la prohibición explícita del enunciado.

### D9. Estado del sistema y habilitación progresiva

`system_state` en MongoDB es la fuente de verdad. El `api-gateway`:

- Lee `system_state` al arrancar.
- Se suscribe a `state.changed` y `recommendations.ready` para actualizar su caché en memoria.
- Relee `system_state` cada 5 s como respaldo si se pierde un evento.
- Expone `GET /api/system/status` con `curation`, `recommendations` y un campo derivado `features: {catalog, recommendations, interactions}`.

El frontend consulta el estado al cargar y cada 2 s mientras haya procesos pendientes; al pasar a operativo detiene el sondeo y habilita recomendaciones e interacciones sin recargar.

### D10. Ingesta asincrónica y consistencia

`interaction-service` valida rango y formato, consulta al `catalog-service` por existencia de película y usuario, y publica con publisher confirms. Responde `202 Accepted` con un `interactionId`.

`interaction-worker` consume con ack manual, hace `upsert` por `(user_id, movie_id)` y publica `interaction.created`. Un índice único garantiza la idempotencia aunque se reentregue el mensaje. Los fallos transitorios se reintentan con backoff; al agotar intentos pasan a `interactions.dlq`.

### D11. Frontend: arquitectura de información, identidad visual y auditoría

Rutas: `/` (home con hero, recomendaciones y catálogo), `/movies` (exploración completa con búsqueda, filtros y paginación), `/movies/:id` (detalle + similares + reseñas + formulario) y `/audit` (fuera de la navegación pública). La identidad activa se guarda en `localStorage`. Se conserva el alcance funcional previo: el catálogo y las recomendaciones siguen disponibles en `/`; `/movies` agrega una vista dedicada de exploración.

`/audit` consume `summary` y `entries` del gateway, muestra tarjetas de métricas, filtros por tipo y tablas paginadas con antes/después y justificación del LLM.

#### D11.1 Referencias y criterio de adaptación

`docs/imgs_ejemplo/inicio.png`, `movies.png` y `banner.png` son la referencia de tono, jerarquía, densidad y composición del frontend. No son un contrato pixel-perfect: se adaptan al contenido real del dataset y a las capacidades existentes.

- Se toma el lenguaje visual (fondo oscuro, títulos display en mayúsculas, acento violeta, tarjetas con scrim, chips tipo pill y grilla de posters), no los textos, títulos ni posters ficticios de las imágenes.
- No se incorporan series, favoritos ni reproducción de video: la navegación queda en **Inicio** y **Películas**, y el CTA principal dice **Ver ficha**. El CTA secundario lleva a las reseñas.
- Las referencias no se sirven como assets de la aplicación; solo se usan durante el diseño y la revisión visual.

#### D11.2 Paleta y tokens

La paleta base son los cuatro colores aprobados. No se agregan colores de marca; las superficies y bordes se derivan con mezclas y transparencias sobre `#1E1E1E`.

| Token | Valor | Uso |
|---|---|---|
| `--color-bg` | `#1E1E1E` | Fondo global, header, footer y superficies base. |
| `--color-accent` | `#732EE1` | CTA primaria, nav activa, chip activo, badge, foco y controles del carrusel. |
| `--color-text` | `#D8D8D8` | Texto principal, títulos, metadatos legibles y texto sobre imágenes. |
| `--color-muted` | `#656565` | Bordes, divisores, iconos secundarios y estados deshabilitados. |

- Superficies elevadas: `color-mix(in srgb, #D8D8D8 6%, #1E1E1E)`.
- Bordes: `color-mix(in srgb, #656565 40%, transparent)`.
- Scrims: degradados de `#1E1E1E` con alfa para garantizar legibilidad sobre backdrops y posters.
- El acento se reserva para acciones y estados activos; no se usa como fondo de secciones extensas.
- `#656565` no se usa para texto informativo: su contraste sobre `#1E1E1E` es insuficiente. Los metadatos secundarios usan `#D8D8D8` al 75 %, y `#656565` queda para bordes y elementos deshabilitados.

#### D11.3 Tipografía

- **Inter variable**, empaquetada con `@fontsource-variable/inter` y sin CDN; fallback `system-ui, "Segoe UI", sans-serif`.
- Títulos display en mayúsculas, peso 800, `letter-spacing: -0.01em`, `line-height: 0.95` y tamaño `clamp(2.75rem, 6vw, 4.5rem)`.
- Título de página 2.25rem/700; título de sección 1.75rem/700; título de tarjeta 1rem/600; cuerpo 1rem/1.6; metadatos 0.875rem; eyebrow 0.8125rem en mayúsculas con `letter-spacing: 0.18em`.
- Años, duraciones y ratings usan `font-variant-numeric: tabular-nums`.

#### D11.4 Layout, grilla y responsive

- Contenedor máximo de 1680 px con padding `clamp(1rem, 4vw, 4rem)`.
- Header de 88 px en desktop y 64 px en mobile, sticky, con fondo `#1E1E1E` y borde inferior sutil al hacer scroll.
- Hero de `min(82vh, 760px)` con contenido alineado a la izquierda y ancho máximo de 560 px.
- Grilla de posters: 6 columnas ≥1536 px, 5 ≥1280 px, 4 ≥1024 px, 3 ≥768 px y 2 por debajo; `gap` de 1.5rem; posters con relación 2:3 y radio de 12 px.
- Tiles de género: 4 columnas ≥1024 px, 2 ≥640 px y 1 por debajo; relación aproximada 2.4:1.
- Ritmo vertical de 4rem entre secciones en desktop y 2.5rem en mobile.
- Breakpoints de Tailwind: `sm` 640, `md` 768, `lg` 1024, `xl` 1280 y `2xl` 1536.

#### D11.5 Componentes base

- `Button`: primario (fondo `#732EE1`, texto `#D8D8D8`), secundario (fondo `#D8D8D8`, texto `#1E1E1E`) y fantasma; altura 48 px, radio 6 px y foco visible de 2 px en `#732EE1`.
- `Chip`: pill de 36 px, borde `#656565` y activo con fondo `#732EE1`; se usa para filtros y para el selector de idioma.
- `Badge`: `NEW`, clasificación por edad, calidad y rating cuando el dato exista; contorno o fondo de acento según el caso. Solo se muestra `NEW` cuando el dato de novedad exista; no se inventa.
- `PosterCard`: imagen con scrim inferior, título superpuesto en mayúsculas y metadatos debajo (año • género); hover sutil y foco visible.
- `SectionHeader`: título, subtítulo opcional y enlace “Ver todo →”.
- `Carousel`: controles anterior/siguiente, indicador numérico y barras; navegación por teclado; sin autoplay por defecto. Si se habilita, dura 8 s, se pausa al interactuar y se desactiva con `prefers-reduced-motion`.
- `UserMenu`: pill con avatar, identificador activo y chevron; despliega el selector buscable de `user-identity`.
- `SearchField`, `StatusBanner`, `Skeleton`, `EmptyState`, `ErrorState`, `ReviewForm`, `ChatPanel` y `Footer`.

#### D11.6 Composición de pantallas

**Home `/`** — header, hero con backdrop, eyebrow (“TU PRÓXIMA OBSESIÓN” o “RECOMENDADO PARA VOS”), título display, fila de metadatos (año, clasificación, duración, géneros y calidad cuando el dato exista), sinopsis de 2–3 líneas y CTAs “Ver ficha” / “Ver reseñas”. Indicadores de carrusel abajo a la izquierda y etiqueta de contexto abajo a la derecha. Debajo, el riel “Recomendadas para vos” (personalizado o respaldo por popularidad); se oculta si no tiene datos. Luego la sección “Según tu estado de ánimo” con cuatro tiles de género que enlazan a `/movies?genre=...`, y el footer.

**Exploración `/movies`** — título “Encontrá tu próxima favorita”, subtítulo, chips de filtro por género y grilla de posters. Cada tarjeta muestra el título sobre la imagen y, debajo, título y metadatos. Paginación accesible al pie. La búsqueda y el filtro se reflejan en la URL para poder compartir el estado.

**Detalle `/movies/:id`** — backdrop con scrim, poster, título, metadatos, sinopsis corta y extendida, reparto y director. CTA primario “Escribir reseña” (ancla al formulario) y secundario “Ver similares” (ancla al riel). Formulario de voto y reseña, listado paginado de reseñas y riel de similares.

**Auditoría `/audit`** — mismo lenguaje visual, sin enlace en la navegación. Tarjetas de métricas con números en acento, chips de filtro por tipo, tabla de antes/después y justificación del LLM; en mobile la tabla se convierte en tarjetas apiladas.

**Asistente y traducción** — el chat cinéfilo se abre en un panel lateral con burbujas, historial, campo de entrada y estados de carga/error; el selector de idioma usa un chip en el header o en la ficha. Ambos reutilizan los tokens y no bloquean el resto de la navegación.

**Estados** — skeletons para hero, tarjetas y rieles; estados vacíos con mensaje y acción; errores con reintento; imágenes faltantes con placeholder de marca; controles deshabilitados con opacidad reducida y motivo visible. El banner de estado informa curación e indexación y se anuncia con `aria-live`.

#### D11.7 Accesibilidad

- Landmarks semánticos, enlace “Saltar al contenido”, foco visible en `#732EE1` y navegación completa por teclado.
- Objetivos táctiles de al menos 44 px; sin desborde horizontal desde 320 px.
- `alt` descriptivo en posters y backdrops; `aria-label` en iconos y controles del carrusel.
- Contraste mínimo AA para texto informativo; `#656565` solo en bordes y estados deshabilitados.
- Respeto de `prefers-reduced-motion` para carruseles, hover y transiciones.

#### D11.8 Assets y fallbacks

- Posters y backdrops salen de `movies.poster`; se cargan con `loading="lazy"`, `decoding="async"` y dimensiones explícitas para evitar saltos de layout.
- Si falta la imagen o falla la carga, se muestra un placeholder con el título y fondo `#656565`, sin icono de imagen rota.
- Iconos con `lucide-react` y tipografía empaquetada; sin dependencias de CDN en tiempo de ejecución.
- Las imágenes de `docs/imgs_ejemplo/` no se incluyen en el bundle.

#### D11.9 Verificación visual

- Revisión en 1440×900, 1024×768 y 390×844 contra las tres referencias, con checklist de header, hero, chips, grilla, tiles, footer y estados.
- Las diferencias deliberadas (navegación sin Series/Mi lista, CTA “Ver ficha” y chips de género en lugar de tipos de contenido) se documentan en `docs/frontend.md`.
- Pruebas de componentes para estados de carga, vacío, error, imagen faltante y deshabilitado.
- Revisión de accesibilidad con axe sin violaciones críticas en `/`, `/movies`, `/movies/:id` y `/audit`.

### D12. Inicialización del dataset

`dataset/` contiene los archivos semilla y un script `init-mongo.sh` montado en `/docker-entrypoint-initdb.d/`. El script usa `mongoimport` para cargar `movies`, `reviews` y `users`, y luego crea índices. Solo se ejecuta con el volumen vacío. La carga es idempotente por volumen, no por script.

### D13. Pruebas y verificación

- **Unitarias**: reglas de curación con fixtures de anomalías; validadores; precomputación de recomendaciones.
- **Integración**: compose completo con cold start, transición de estados, flujo de interacción y `/audit`.
- **Aislamiento de red**: `docker compose exec frontend getent hosts mongodb` debe fallar.
- **Latencia**: script de carga que mide p95 de recomendaciones (<100 ms).
- **Aceptación**: `docker compose down -v && docker compose up --build` y recorrido manual del flujo completo.

## Risks / Trade-offs

- **Costo/tiempo del LLM durante la curación** → lotes, concurrencia limitada, caché, timeout y modo degradado con pendientes.
- **Descarga inicial del modelo Ollama** → volumen persistente, reintentos y proveedor API alternativo por entorno.
- **Calidad de recomendaciones con matriz dispersa** → híbrido contenido + colaborativo y baseline para usuarios sin historial.
- **Cold start largo por 20k películas y 61k reseñas** → fases idempotentes, métricas de progreso y warm start sin recómputo.
- **Acoplamiento por base compartida** → módulos `repository` por servicio, escrituras limitadas al dueño de cada colección.
- **Duplicados históricos de votos** → índice único creado después de consolidar, no antes.
- **Pérdida de eventos de estado** → fuente de verdad en MongoDB más relectura periódica en el gateway.
- **Imagen de Ollama pesada** → no se publica al host y el modelo vive en volumen; se documenta el requisito de internet en el primer arranque.
- **Dataset con esquema inesperado** → fase 0 de inspección y adaptadores de importación antes de fijar contratos definitivos.
- **Fidelidad visual vs. datos reales** → se adapta la composición de las referencias; no se inventan series, favoritos, calidad 4K ni badges de novedad.
- **Posters faltantes o remotos** → placeholder de marca, `loading="lazy"` y dimensiones explícitas para evitar saltos de layout.
- **Contraste de la paleta** → `#656565` se limita a bordes y estados deshabilitados; los metadatos usan `#D8D8D8` al 75 %.

## Migration Plan

Proyecto greenfield: no hay migración de datos productivos.

1. Implementar en el orden de `tasks.md` (fundaciones → catálogo → interacciones → curación → recomendaciones → ciclo de vida → IA → entrega).
2. Cada fase deja el compose levantable y verificable.
3. Rollback de datos: `docker compose down -v` restaura el estado inicial; los volúmenes se recrean desde el dataset semilla.
4. Rollback de código: revertir el commit de la fase; las fases no comparten migraciones destructivas.

## Open Questions

- El esquema y los nombres exactos de los archivos del dataset se confirmarán al incorporarlo; si difieren, se ajusta solo el importador y los mapeos, sin cambiar las specs.
- El modelo Ollama definitivo se elegirá según el hardware de evaluación; el contrato `LLMProvider` lo desacopla.
- El registro dinámico de usuarios queda fuera de este cambio y podría abordarse en una propuesta posterior.
