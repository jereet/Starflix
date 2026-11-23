# Starflix (DCIC-Flix26)

Plataforma web de catálogo cinematográfico basada en microservicios, ingesta asincrónica con RabbitMQ, sistema de recomendaciones y curación de datos asistida por LLMs.

---

## Estructura del Sistema

* **Frontend**: Cliente web (interfaz de usuario).
* **API Gateway**: Punto único de entrada al backend.
* **Microservicios**:
  * **Catalog Service**: Exploración, búsqueda y detalle de películas.
  * **Interaction Service**: Recepción y encolado de valoraciones y reseñas.
  * **Interaction Worker**: Procesamiento desacoplado y persistencia de interacciones.
  * **Recommendation Service**: Cálculo y entrega de recomendaciones personalizadas y similares.
  * **Cleaning Service**: Detección de anomalías y saneamiento del catálogo con soporte de Clean LLM.
  * **AI Service**: Traducción de contenidos y asistente cinéfilo conversacional.
* **Mensajería**: RabbitMQ (absorción de ráfagas y coordinación de eventos de estado).
* **Persistencia**: MongoDB (`Movies`, `Reviews`, `Audit Logs`) mediado por una capa centralizada de `Repository` y `Repository MCP`.

---

## Documentación de Arquitectura

El detalle de la ingeniería del sistema se encuentra documentado en la carpeta [docs/](docs/README.md):

* [Visión general de arquitectura y diagrama](docs/README.md)
* [Requerimientos y flujo](docs/requerimientos-y-flujo.md)
* [Componentes](docs/componentes.md)
* [Notas y propuestas](docs/notas-y-propuestas.md)

---

## Ejecución del entorno

El sistema completo se orquesta mediante Docker Compose:

```bash
docker compose up --build
```
