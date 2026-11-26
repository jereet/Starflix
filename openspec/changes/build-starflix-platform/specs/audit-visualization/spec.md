# Spec Delta

## Purpose

Permitir que los evaluadores inspeccionen el resultado de la curación mediante una ruta oculta con métricas consolidadas y el detalle de cada registro corregido o inferido.

## ADDED Requirements

### Requirement: Ruta oculta de auditoría

El frontend DEBE exponer una vista de auditoría accesible directamente por la URL `/audit`, sin enlace en la navegación pública.

#### Scenario: Acceso directo a la ruta
- **WHEN** un evaluador navega directamente a `/audit`
- **THEN** el sistema muestra la vista de auditoría

#### Scenario: Navegación pública
- **WHEN** un visitante recorre la barra de navegación o el menú principal
- **THEN** no encuentra un enlace visible hacia `/audit`

### Requirement: Métricas macro de la curación

La vista de auditoría DEBE mostrar un tablero con los totales consolidados de anomalías detectadas y saneadas por tipo.

#### Scenario: Tablero con el proceso finalizado
- **WHEN** el evaluador abre `/audit` después de completada la curación
- **THEN** ve los totales de películas deduplicadas, años normalizados, géneros corregidos, ratings de IMDb ajustados, comentarios basura descartados, reseñas huérfanas eliminadas, votos consolidados y calificaciones imputadas por el LLM

#### Scenario: Tablero durante la curación
- **WHEN** el evaluador abre `/audit` mientras la curación está en curso
- **THEN** ve las métricas parciales acumuladas hasta ese momento
- **AND** el sistema indica que el proceso todavía no finalizó

### Requirement: Detalle de registros modificados

La vista de auditoría DEBE permitir explorar los registros individuales corregidos o depurados, mostrando el valor original, el valor resultante y el motivo de la corrección.

#### Scenario: Inspección de una corrección
- **WHEN** el evaluador selecciona un registro de la auditoría
- **THEN** ve el valor original y el valor corregido
- **AND** ve el motivo registrado por el agente

#### Scenario: Inspección de una inferencia del LLM
- **WHEN** el evaluador selecciona una calificación inferida por el LLM
- **THEN** ve el comentario original, la calificación inferida y la justificación del modelo

### Requirement: Navegación y filtrado del detalle

El detalle de auditoría DEBE poder filtrarse por tipo de anomalía y recorrerse de forma paginada.

#### Scenario: Filtro por tipo de anomalía
- **WHEN** el evaluador selecciona un tipo de anomalía
- **THEN** el detalle muestra solo los registros de ese tipo

#### Scenario: Conjunto grande de registros
- **WHEN** el detalle contiene más registros que los visibles en una página
- **THEN** el evaluador puede avanzar y retroceder entre páginas

### Requirement: Consulta de auditoría mediada por el Gateway

El frontend DEBE obtener las métricas y el detalle de auditoría a través del API Gateway, sin acceder directamente a MongoDB.

#### Scenario: Carga de la vista de auditoría
- **WHEN** el evaluador abre `/audit`
- **THEN** el frontend solicita las métricas y el detalle al API Gateway
- **AND** el Gateway las obtiene del Cleaning Service

### Requirement: Disponibilidad de la auditoría

La vista de auditoría DEBE estar disponible tanto durante como después del proceso de curación, sin requerir autenticación adicional.

#### Scenario: Auditoría antes de la primera curación
- **WHEN** el evaluador abre `/audit` antes de que el agente haya registrado datos
- **THEN** la vista responde con métricas en cero y un mensaje que indica que aún no hay registros
