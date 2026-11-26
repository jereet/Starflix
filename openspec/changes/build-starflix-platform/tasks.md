# Tasks

## 1. Fundaciones e infraestructura

- [ ] 1.1 Inspeccionar `dataset/` y documentar el esquema real (nombres de archivos, campos y tipos) en `docs/dataset.md`; si el dataset aún no está, documentar el contrato esperado y crear fixtures reducidos en `tests/fixtures/`. Verificación: el documento lista cada colección y sus campos, y los fixtures cargan en una base temporal.
- [ ] 1.2 Crear la estructura del monorepo (`frontend/`, `services/`, `infra/`, `dataset/`, `tests/`) con un README por servicio. Verificación: `tree`/listado muestra la estructura y cada servicio tiene su `README.md`.
- [ ] 1.3 Implementar `services/shared` con configuración por entorno, conexión a MongoDB, conexión a RabbitMQ, logging estructurado, manejo de errores y utilidades de fecha. Verificación: tests unitarios de configuración y errores pasan con `pytest services/shared`.
- [ ] 1.4 Crear `docker-compose.yml` base con MongoDB y RabbitMQ usando imágenes oficiales, volúmenes persistentes, healthchecks y las redes `frontend_net`, `backend_net` y `data_net`. Verificación: `docker compose config` es válido y `docker compose up -d mongodb rabbitmq` deja ambos servicios saludables.
- [ ] 1.5 Agregar el script de inicialización en `dataset/init-mongo.sh` montado en `/docker-entrypoint-initdb.d/`, con `mongoimport` de las colecciones e índices base. Verificación: sobre un volumen vacío, `docker compose up -d mongodb` carga los conteos esperados y los índices existen.
- [ ] 1.6 Crear el scaffold del frontend con React, Vite, TypeScript, Tailwind y React Router, más su Dockerfile con Nginx. Incluir el tema de Tailwind con los tokens de `design.md` (paleta `#1E1E1E`, `#732EE1`, `#D8D8D8` y `#656565`), la tipografía Inter empaquetada y el shell de navegación (header, footer, contenedor y fondo global). Verificación: `npm run build` compila, el contenedor sirve el shell en el puerto publicado y no hay peticiones a CDNs externos.
- [ ] 1.7 Crear el esqueleto del `api-gateway` en FastAPI con `/health`, manejo uniforme de errores y un `GET /api/system/status` provisional. Verificación: `docker compose up -d api-gateway` y `curl localhost:<puerto>/health` responden 200.
- [ ] 1.8 Documentar el sistema de diseño del frontend en `docs/frontend.md`: tokens, tipografía, espaciado, componentes base, breakpoints, estados y criterios de adaptación de `docs/imgs_ejemplo/`. Verificación: el documento permite construir una pantalla nueva sin inventar colores ni medidas y enumera las diferencias deliberadas respecto de las referencias.

## 2. Catálogo e identidad

- [ ] 2.1 Implementar el repositorio y `GET /internal/movies` en `catalog-service` con paginación y metadatos de página. Verificación: tests de integración contra MongoDB devuelven la primera página y una página vacía al final.
- [ ] 2.2 Implementar búsqueda por título y filtro por género combinables. Verificación: tests cubren coincidencia parcial sin distinguir mayúsculas, filtro simple y combinación de ambos.
- [ ] 2.3 Implementar `GET /internal/movies/{id}` con todos los campos y respuesta 404 para inexistentes. Verificación: tests de película existente e inexistente pasan.
- [ ] 2.4 Implementar `GET /internal/movies/{id}/reviews` paginado y ordenado por fecha descendente. Verificación: tests con película con y sin reseñas pasan.
- [ ] 2.5 Implementar `GET /internal/users` con listado y búsqueda por identificador. Verificación: tests devuelven los usuarios sembrados y filtran por prefijo.
- [ ] 2.6 Exponer en el gateway `GET /api/movies`, `/api/movies/{id}`, `/api/movies/{id}/reviews` y `/api/users`. Verificación: tests de contrato del gateway validan códigos, forma de respuesta y ruteo al servicio correcto.
- [ ] 2.7 Construir la vista de exploración `/movies` con grilla de posters, paginación, búsqueda y filtros por género, siguiendo la composición de `docs/imgs_ejemplo/movies.png`: encabezado con título y subtítulo, chips de filtro, tarjetas con título superpuesto y metadatos debajo, y grilla responsive de 6/4/2 columnas. Verificación: tests de componentes cubren render, búsqueda, cambio de filtro y estado vacío; la vista funciona contra el gateway local y no desborda a 390 px.
- [ ] 2.8 Construir la ficha de detalle con backdrop, poster, sinopsis corta y extendida, reparto, director y listado navegable de reseñas, reutilizando los tokens y componentes del sistema de diseño. Verificación: tests de componentes cubren película con datos completos, película sin poster (placeholder) y película inexistente.
- [ ] 2.9 Implementar el selector de identidad con búsqueda y persistencia en `localStorage`, integrado en el header como pill de usuario con avatar, identificador activo y menú desplegable. Verificación: test de recarga mantiene el usuario activo, el menú es navegable por teclado y el envío de acciones usa ese identificador.
- [ ] 2.10 Documentar los endpoints de catálogo y usuarios en `docs/api.md` con ejemplos ejecutables. Verificación: cada ejemplo documentado responde como se describe contra el compose local.
- [ ] 2.11 Construir la home `/` con hero de películas destacadas, carrusel accesible y sección “Según tu estado de ánimo” con tiles de género que enlazan a `/movies?genre=...`, siguiendo el tono de `docs/imgs_ejemplo/inicio.png` y `banner.png`. Verificación: tests de componentes cubren hero con y sin datos, navegación del carrusel y enlace de cada tile al filtro correcto.

## 3. Ingesta asincrónica de interacciones

- [ ] 3.1 Implementar en `services/shared` la topología de RabbitMQ (exchange `starflix.events`, colas `interactions`, `interactions.dlq`, `recommendation.refresh`, `state.changed`) con declaración idempotente y mensajes persistentes. Verificación: test de declaración ejecutado dos veces no falla y las colas existen con sus bindings.
- [ ] 3.2 Implementar `POST /internal/interactions` en `interaction-service`: validación de rango, verificación de existencia de película y usuario, publicación con publisher confirms y respuesta 202. Verificación: tests cubren éxito, rating inválido, película inexistente y usuario inexistente.
- [ ] 3.3 Implementar `interaction-worker` con consumo manual, `upsert` por `(user_id, movie_id)`, ack, reintentos con backoff y derivación a DLQ. Verificación: tests cubren persistencia, reentrega sin duplicados y mensaje irrecuperable en DLQ.
- [ ] 3.4 Publicar el evento `interaction.created` después de persistir y consumirlo en una cola de prueba. Verificación: test de integración confirma que el evento se emite una vez por interacción persistida.
- [ ] 3.5 Exponer `POST /api/interactions` en el gateway con validación de borde y bloqueo cuando el sistema no está listo. Verificación: tests de contrato cubren 202, 400 y 423.
- [ ] 3.6 Agregar el formulario de voto y reseña en la ficha de película, con estado pendiente y confirmación. Verificación: test de componente envía rating y comentario y muestra el estado de encolado.
- [ ] 3.7 Escribir un test de integración de ráfaga: publicar 100 interacciones y verificar que todas se persisten sin duplicados y que la reseña aparece en el catálogo. Verificación: el test termina en verde contra el compose.
- [ ] 3.8 Documentar el flujo asincrónico y la topología de colas en `docs/interactions.md`. Verificación: el documento incluye el diagrama de flujo y los comandos para inspeccionar las colas.

## 4. Curación de datos y auditoría

- [ ] 4.1 Implementar las reglas de películas: deduplicación por título normalizado y año, normalización de año, duración, géneros, rating de IMDb y espacios en títulos, con cuarentena para duplicados. Verificación: tests unitarios con fixtures de cada anomalía producen el valor esperado y registran la corrección.
- [ ] 4.2 Implementar las reglas de reseñas: spam, fechas dispares, huérfanas y votos duplicados. Verificación: tests unitarios cubren cada regla y el conteo de consolidación.
- [ ] 4.3 Implementar el registro de auditoría con `anomalyType`, `documentId`, `before`, `after`, `reason`, datos del LLM e índices de consulta. Verificación: test confirma que cada corrección de las reglas 4.1 y 4.2 genera un registro consultable.
- [ ] 4.4 Implementar `LLMProvider` con adaptadores `ollama` y `openai-compatible`, y agregar el contenedor Ollama al compose con volumen de modelos. Verificación: test con proveedor falso valida la interfaz y `docker compose up -d ollama` responde en su endpoint de salud.
- [ ] 4.5 Implementar la inferencia de rating: selección de reseñas con `rating: null`, prompt JSON, validación entera 1–5, reintentos, caché por hash y degradación a pendiente. Verificación: tests cubren respuesta válida, inválida, reintento, caché y proveedor caído.
- [ ] 4.6 Orquestar el pipeline de curación con lock con heartbeat, fases, `curationVersion`, métricas y escritura de `system_state`. Verificación: test de integración ejecuta dos veces seguidas y la segunda no reprocesa documentos ya curados.
- [ ] 4.7 Publicar `catalog.cleaned` y `state.changed` al finalizar, incluyendo `dataVersion` y métricas. Verificación: test de integración confirma la recepción de ambos eventos y el estado `done` en MongoDB.
- [ ] 4.8 Implementar `GET /internal/audit/summary` y `GET /internal/audit/entries` con filtro por tipo y paginación. Verificación: tests devuelven métricas en cero sin datos y métricas correctas después de una curación de fixture.
- [ ] 4.9 Construir la ruta oculta `/audit` con tarjetas de métricas, filtros y tablas de detalle con antes/después y justificación del LLM, usando el mismo sistema de diseño y convirtiendo la tabla en tarjetas apiladas en mobile. Verificación: tests de componentes cubren estado vacío, estado con datos, paginación y vista mobile; la ruta no aparece en la navegación.
- [ ] 4.10 Documentar las reglas de curación, el uso del LLM y el contrato de auditoría en `docs/curation.md`. Verificación: el documento enumera cada anomalía, su regla y el campo de auditoría asociado.

## 5. Motor de recomendaciones

- [ ] 5.1 Implementar la carga de datos y el modelo de contenido: metadata soup, TF-IDF, similitud coseno y top-20 de similares por película. Verificación: tests verifican que una película no se incluye a sí misma y que los similares comparten género o metadatos esperados.
- [ ] 5.2 Implementar el modelo colaborativo con SVD o KNN de `scikit-surprise` y el baseline de rating ponderado. Verificación: tests con matriz de juguete producen scores deterministas y el baseline ordena por rating ponderado.
- [ ] 5.3 Implementar la etapa de dos pasos y precomputar top-50 personalizado por usuario y top-20 de similares, persistiendo artefactos en el volumen `recommendation_data`. Verificación: test genera artefactos, los recarga y obtiene los mismos resultados.
- [ ] 5.4 Implementar `GET /internal/recommendations?userId` y `GET /internal/movies/{id}/similar` con exclusión de películas ya vistas y respuesta condicionada al estado. Verificación: tests cubren usuario con historial, usuario sin historial, película sin datos y sistema no listo.
- [ ] 5.5 Consumir `catalog.cleaned`, construir el índice inicial y publicar `recommendations.ready`. Verificación: test de integración arranca con catálogo curado de fixture y observa la transición `indexing` → `ready`.
- [ ] 5.6 Consumir `interaction.created` y refrescar en segundo plano solo los usuarios y la película afectados. Verificación: test confirma que el resto de los usuarios conserva su versión y el afectado recibe una nueva.
- [ ] 5.7 Exponer `GET /api/recommendations` y `GET /api/movies/{id}/similar` en el gateway. Verificación: tests de contrato validan ruteo, parámetros y errores 503/423 cuando no está listo.
- [ ] 5.8 Agregar los rieles “Recomendadas para vos” y “Películas similares” en la home y en la ficha, con scroll horizontal, flechas accesibles y tarjetas del sistema de diseño. Verificación: tests de componentes cubren carga, lista vacía, estado deshabilitado y navegación por teclado.
- [ ] 5.9 Escribir un script de carga que mida el percentil 95 de latencia de recomendaciones con el sistema listo. Verificación: el reporte muestra p95 menor a 100 ms y el script falla si lo supera.
- [ ] 5.10 Documentar la estrategia híbrida, su justificación y los resultados de latencia en el `README.md`. Verificación: la sección describe contenido, colaborativo, baseline, dos etapas y el número medido.

## 6. Ciclo de vida y disponibilidad progresiva

- [ ] 6.1 Definir el modelo `system_state` y sus transiciones (`curating`, `curated`, `indexing`, `ready`, `degraded`) con validación. Verificación: tests unitarios rechazan transiciones inválidas y aceptan la secuencia de arranque en frío.
- [ ] 6.2 Implementar en el gateway la lectura inicial de `system_state`, el consumidor de `state.changed`/`recommendations.ready` y la relectura periódica de respaldo. Verificación: test simula un evento perdido y el gateway se recupera dentro del intervalo configurado.
- [ ] 6.3 Verificar la secuencia completa de arranque en frío con un test de integración: dataset crudo → curación → indexación → `ready`. Verificación: el test observa las transiciones y que el frontend recibe `features` habilitadas al final.
- [ ] 6.4 Implementar la detección de arranque en caliente y el omitir curación e indexación completa cuando `dataVersion` coincide. Verificación: test reinicia el compose sobre el mismo volumen y el tiempo hasta `ready` es menor al del arranque en frío.
- [ ] 6.5 Aplicar el bloqueo de interacciones y recomendaciones en el gateway y en los servicios según el estado. Verificación: tests confirman 423 antes de `ready` y 200/202 después.
- [ ] 6.6 Agregar el banner de estado y el sondeo con desbloqueo automático en el frontend, sin recarga manual, con el estilo de aviso del sistema de diseño y `aria-live`. Verificación: test con estados simulados muestra el mensaje correcto, lo anuncia a lectores de pantalla y habilita los controles al pasar a `ready`.
- [ ] 6.7 Escribir el test de aislamiento de red: desde el contenedor del frontend, `getent hosts mongodb` y una conexión TCP a MongoDB deben fallar. Verificación: el script falla si el frontend resuelve o alcanza MongoDB.
- [ ] 6.8 Documentar el ciclo de vida, los estados y la matriz de funcionalidades por estado en `docs/lifecycle.md`. Verificación: la matriz coincide con los escenarios de `system-lifecycle`.

## 7. Experiencias con IA (deseable)

- [ ] 7.1 Implementar `POST /internal/ai/translate` con selección de idioma, preservación de nombres propios y caché en `translations`. Verificación: tests cubren traducción nueva, segunda llamada cacheada y falla del proveedor.
- [ ] 7.2 Agregar el selector de idioma y la traducción de sinopsis y reseñas en el frontend, con el estilo de chip/pill del sistema de diseño. Verificación: test de componente cambia el idioma y muestra el texto traducido sin alterar rating ni autor.
- [ ] 7.3 Implementar `POST /internal/ai/chat` con recuperación de películas del catálogo y prompt fundamentado. Verificación: tests cubren consulta con criterios, comparación y película inexistente sin datos inventados.
- [ ] 7.4 Construir la interfaz de chat cinéfilo con historial de conversación en un panel lateral, con burbujas y estados de carga/error del sistema de diseño. Verificación: test de componente envía un mensaje, muestra la respuesta y conserva el historial.
- [ ] 7.5 Implementar la degradación controlada de traducción y chat cuando el LLM no responde. Verificación: tests con proveedor caído muestran el mensaje de indisponibilidad y el resto del catálogo sigue operativo.
- [ ] 7.6 Documentar las integraciones de IA y sus variables de entorno en `docs/ai.md`. Verificación: el documento permite cambiar de proveedor sin tocar código y los ejemplos funcionan.

## 8. Integración y entrega

- [ ] 8.1 Ejecutar la aceptación de arranque en frío desde un clon limpio con `docker compose down -v && docker compose up --build` y registrar tiempos y resultado. Verificación: el sistema llega a `ready`, el catálogo es navegable y `/audit` muestra métricas.
- [ ] 8.2 Ejecutar la aceptación de arranque en caliente y registrar el tiempo hasta `ready` con el frontend habilitado desde el inicio. Verificación: no se repite la limpieza pesada y el tiempo es menor al de 8.1.
- [ ] 8.3 Verificar manualmente el flujo completo: elegir usuario, explorar, ver recomendaciones, votar, ver la reseña aparecer y consultar el detalle en `/audit`. Verificación: cada paso se completa sin errores en la consola del navegador ni en los logs.
- [ ] 8.4 Completar el `README.md` con ejecución, diagrama y justificación de redes, estrategia de curación, uso del LLM, `/audit`, estrategia de recomendaciones, integraciones de IA e identidad visual del frontend. Verificación: un evaluador puede seguir el README sin información faltante.
- [ ] 8.5 Ejecutar lint, formateo y chequeo de tipos en todos los servicios y en el frontend. Verificación: los comandos configurados terminan sin errores.
- [ ] 8.6 Ejecutar `openspec validate build-starflix-platform` y la suite de tests completa contra el compose limpio. Verificación: la validación es correcta y todos los tests pasan.
- [ ] 8.7 Ejecutar la revisión visual del frontend contra `docs/imgs_ejemplo/inicio.png`, `movies.png` y `banner.png` en 1440×900, 1024×768 y 390×844, registrando en `docs/frontend.md` las diferencias deliberadas. Verificación: la checklist cubre header, hero, chips, grilla, tiles de género, footer y estados; no hay desbordes horizontales ni imágenes rotas.
- [ ] 8.8 Ejecutar la revisión de accesibilidad del frontend: contraste, foco visible, navegación por teclado, etiquetas ARIA, textos alternativos y `prefers-reduced-motion`. Verificación: axe no reporta violaciones críticas en `/`, `/movies`, `/movies/:id` y `/audit`, y el recorrido completo se puede hacer solo con teclado.

## Workflow follow-up

- Revisar el plan con el equipo antes de implementar.
- Archivar el cambio con `openspec archive build-starflix-platform` cuando la implementación esté verificada.
- Sincronizar las specs principales si se decide archivar sin ejecutar el archivado completo.
