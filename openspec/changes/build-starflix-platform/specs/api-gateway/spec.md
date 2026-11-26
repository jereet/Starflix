# Spec Delta

## Purpose

Concentrar el acceso al backend en un único punto de entrada que enruta solicitudes, valida datos y media toda consulta del frontend hacia los servicios y la base de datos.

## ADDED Requirements

### Requirement: Punto único de entrada al backend

El frontend DEBE comunicarse exclusivamente con el API Gateway. Ningún otro servicio backend DEBE ser accesible directamente desde el navegador.

#### Scenario: Solicitud del frontend
- **WHEN** el frontend necesita catálogo, recomendaciones, interacciones o auditoría
- **THEN** envía la solicitud al API Gateway
- **AND** el Gateway la reenvía al servicio correspondiente

#### Scenario: Acceso directo a un servicio interno
- **WHEN** un cliente intenta alcanzar un microservicio backend sin pasar por el Gateway
- **THEN** la solicitud no obtiene respuesta funcional desde la red del frontend

### Requirement: Ruteo por dominio funcional

El API Gateway DEBE enrutar cada solicitud al servicio responsable: catálogo, interacciones, recomendaciones, curación y auditoría, o experiencias con LLM.

#### Scenario: Ruteo de una consulta de catálogo
- **WHEN** llega una solicitud de exploración, búsqueda o detalle de películas
- **THEN** el Gateway la deriva al Catalog Service

#### Scenario: Ruteo de una nueva interacción
- **WHEN** llega una solicitud de voto y reseña
- **THEN** el Gateway la deriva al Interaction Service

#### Scenario: Ruteo de una consulta de auditoría
- **WHEN** llega una solicitud a la ruta de auditoría
- **THEN** el Gateway la deriva al Cleaning Service

### Requirement: Validación de solicitudes en el borde

El API Gateway DEBE validar la forma y los rangos de los datos de entrada antes de derivar una solicitud, rechazando las inválidas con un error de cliente.

#### Scenario: Voto fuera de rango
- **WHEN** llega un voto con una calificación menor a 1 o mayor a 5
- **THEN** el Gateway responde con un error de solicitud inválida
- **AND** no publica ningún mensaje en RabbitMQ

#### Scenario: Identificador de usuario inexistente
- **WHEN** una solicitud referencia un usuario fuera del conjunto `usr_0001` a `usr_2000`
- **THEN** el Gateway responde con un error de solicitud inválida

### Requirement: Exposición del estado del sistema

El API Gateway DEBE exponer un endpoint de estado que informe el estado de curación del catálogo y de inicialización del recomendador, y DEBE conservar el último estado conocido aunque los eventos ya hayan sido consumidos.

#### Scenario: Consulta del estado durante el arranque en frío
- **WHEN** el frontend consulta el estado del sistema mientras la curación está en curso
- **THEN** el Gateway responde que la curación está en progreso y las recomendaciones no están disponibles

#### Scenario: Estado tras la habilitación plena
- **WHEN** la curación y la indexación del recomendador finalizaron
- **THEN** el Gateway responde que el sistema está plenamente operativo
- **AND** sigue informando ese estado en consultas posteriores sin depender de mensajes retenidos en RabbitMQ

### Requirement: Manejo de fallas de servicios internos

Cuando un servicio interno no responde o lo hace con error, el API Gateway DEBE devolver una respuesta de error controlada sin exponer detalles internos ni dejar la solicitud colgada.

#### Scenario: Servicio de recomendaciones no disponible
- **WHEN** el Recommendation Service no responde dentro del tiempo límite
- **THEN** el Gateway responde con un error de servicio no disponible
- **AND** el frontend puede mostrar un mensaje de reintento al usuario
