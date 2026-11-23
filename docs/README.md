# Arquitectura del Sistema

Visión general y diseño de componentes de la plataforma Starflix (DCIC-Flix26), basada en microservicios, procesamiento asincrónico por eventos y agentes de IA.

## Diagrama de Arquitectura

![Diagrama de Arquitectura](architecture-diagram.png)

> Archivo fuente editable disponible en [Starflix architecture.drawio](Starflix%20architecture.drawio).

---

## Flujo general de componentes

* **Frontend y Gateway**:
  * `Frontend → API Gateway`
  * `API Gateway → Recommendation Service`
  * `API Gateway → Catalog Service`
  * `API Gateway → AI Service`
  * `API Gateway → Cleaning Service`
  * `API Gateway → RabbitMQ`
  * `API Gateway → Interaction Service`

* **Catálogo e IA**:
  * `Catalog Service → AI Service → LLM`
  * `Catalog Service → Repository`

* **Recomendaciones**:
  * `Recommendation Service → Repository`

* **Curación y Auditoría**:
  * `Cleaning Service → Clean LLM → Repository MCP → Repository`
  * `Cleaning Service → Repository`
  * `Cleaning Service → RabbitMQ`

* **Interacciones asincrónicas**:
  * `Interaction Service → RabbitMQ`
  * `RabbitMQ ⇄ Interaction Worker`

* **Persistencia centralizada**:
  * `Repository → MongoDB (Movies, Reviews, Audit Logs)`

---

## Índice de Documentación

* [Requerimientos y flujo](requerimientos-y-flujo.md): Mapeo de flujos técnicos asociados a cada requerimiento del sistema.
* [Componentes](componentes.md): Detalle y responsabilidades de cada microservicio, cliente, broker y repositorios.
* [Notas y propuestas](notas-y-propuestas.md): Consideraciones y propuestas técnicas del equipo.
