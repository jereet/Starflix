# Proposal

## Why

La cátedra de Tópicos Avanzados de Desarrollo Web exige entregar **DCIC-Flix26 / Starflix** antes del 08/12/2026: una plataforma de catálogo cinematográfico con microservicios, ingesta asincrónica, curación de datos con LLM y recomendaciones de baja latencia, ejecutable con un único `docker compose up --build`. El equipo ya cuenta con un diseño arquitectónico en `docs/` y el enunciado en `Enunciado.md`, pero todavía no existe una especificación de comportamiento acordada ni un plan de implementación por etapas. Este cambio crea esa especificación y ordena el desarrollo en pasos verificables.

## What Changes

- Se define el comportamiento observable del sistema completo (catálogo, identidad de usuario, interacciones asincrónicas, recomendaciones, curación, auditoría, ciclo de vida y experiencias con LLM).
- Se establece la topología de despliegue: `docker-compose.yml` con MongoDB y RabbitMQ usando imágenes oficiales, volúmenes persistentes, subredes aisladas y arranque sin pasos manuales.
- Se fija la regla de seguridad de red: el contenedor del frontend no puede resolver ni alcanzar a MongoDB; todo acceso pasa por el API Gateway y los servicios backend.
- Se especifica el ciclo de arranque en frío y en caliente, incluida la disponibilidad progresiva de funcionalidades y la coordinación por eventos en RabbitMQ.
- Se especifican las reglas determinísticas de curación, la inferencia semántica con LLM y el registro de auditoría consultable desde la ruta oculta `/audit`.
- Se especifican las recomendaciones personalizadas y de películas similares con resultados precalculados y latencia objetivo menor a 100 ms.
- Se incluyen como objetivos deseables la traducción multilingüe y el asistente cinéfilo conversacional.
- No se modifica código existente: el repositorio solo contiene documentación y andamiaje de OpenSpec.

## Capabilities

### New Capabilities

- `platform-deployment`: orquestación con Docker Compose, imágenes oficiales de MongoDB y RabbitMQ, volúmenes, subredes aisladas, inicialización del dataset y arranque con un solo comando.
- `api-gateway`: punto único de entrada al backend, ruteo hacia los microservicios y mediación de todo acceso del frontend a los datos.
- `catalog`: exploración, búsqueda, filtrado, detalle de películas y lectura de reseñas existentes.
- `user-identity`: selección de identidad entre los usuarios `usr_0001` a `usr_2000` y actuación en su nombre.
- `interaction-ingestion`: emisión unificada de voto y reseña, publicación en RabbitMQ y persistencia desacoplada por un worker.
- `recommendations`: recomendaciones personalizadas y de películas similares, precalculadas y servidas con baja latencia.
- `data-curation`: detección y corrección determinística de anomalías, inferencia semántica con LLM y registro de auditoría.
- `audit-visualization`: ruta oculta `/audit` con métricas macro y detalle de registros corregidos o inferidos.
- `system-lifecycle`: arranque en frío y en caliente, estado de curación e indexación, y habilitación progresiva de funcionalidades.
- `ai-experience`: traducción dinámica de contenidos y asistente cinéfilo conversacional como capacidades deseables.

### Modified Capabilities

<!-- Sin capacidades existentes: el proyecto no tiene specs previas. -->

## Impact

- **Nuevo código**: `docker-compose.yml`, `dataset/`, frontend, API Gateway, Catalog Service, Interaction Service, Interaction Worker, Recommendation Service, Cleaning Service y AI Service.
- **Infraestructura**: MongoDB y RabbitMQ contenerizados, volúmenes persistentes, subredes Docker y scripts de inicialización en `/docker-entrypoint-initdb.d/`.
- **Datos**: colecciones `movies`, `reviews`, `users` y `audit_logs`; dataset semilla provisto por la cátedra.
- **Documentación**: `README.md` técnico con arquitectura de red, estrategia de curación, uso de LLM, ruta `/audit` y justificación del motor de recomendaciones.
- **Dependencias externas**: proveedor de LLM (API o modelo local) y librerías de recomendación (por definir en `design.md`).
- **Sin impacto retrocompatible**: no hay implementación previa que conservar.
