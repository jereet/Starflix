# Spec Delta

## Purpose

Entregar recomendaciones personalizadas y de películas similares a partir de modelos e índices precalculados, con latencia mínima y sin cómputo pesado durante las peticiones del usuario.

## ADDED Requirements

### Requirement: Recomendaciones personalizadas

El sistema DEBE devolver una lista de películas recomendadas para un usuario según su historial de calificaciones y reseñas.

#### Scenario: Usuario con historial
- **WHEN** el frontend solicita recomendaciones para un usuario con interacciones previas
- **THEN** el sistema devuelve una lista acotada de películas ordenadas por afinidad estimada
- **AND** excluye las películas que el usuario ya calificó

#### Scenario: Usuario sin historial suficiente
- **WHEN** el frontend solicita recomendaciones para un usuario sin historial o con historial insuficiente
- **THEN** el sistema devuelve una lista de respaldo basada en popularidad y calificación ponderada
- **AND** informa que se trata de recomendaciones generales

### Requirement: Recomendación de películas similares

El sistema DEBE devolver películas similares a una película dada, basándose en afinidad temática o en patrones de co-calificación.

#### Scenario: Película con vecinos calculados
- **WHEN** el usuario abre la ficha de una película y solicita similares
- **THEN** el sistema devuelve una lista acotada de películas relacionadas
- **AND** no incluye la película consultada dentro de los resultados

#### Scenario: Película sin suficientes datos
- **WHEN** no existen datos suficientes para calcular similares de una película
- **THEN** el sistema devuelve una lista vacía o un respaldo por género sin fallar

### Requirement: Resultados precalculados

El motor de recomendaciones DEBE construir sus modelos e índices fuera del ciclo de una petición HTTP y NO DEBE reentrenar modelos ni calcular similitudes contra todo el catálogo durante una solicitud del usuario.

#### Scenario: Petición de recomendaciones
- **WHEN** llega una solicitud de recomendaciones
- **THEN** el servicio responde consultando estructuras precalculadas en memoria o persistidas
- **AND** no ejecuta entrenamiento ni un recorrido completo del catálogo en esa petición

### Requirement: Latencia objetivo

Los endpoints de recomendaciones DEBEN responder en menos de 100 ms para el percentil 95 bajo carga normal, una vez completada la inicialización.

#### Scenario: Medición de latencia con el sistema listo
- **WHEN** se realizan consultas repetidas de recomendaciones con el sistema plenamente operativo
- **THEN** el percentil 95 de los tiempos de respuesta es inferior a 100 ms

#### Scenario: Sistema aún inicializando
- **WHEN** el recomendador todavía no completó su indexación
- **THEN** responde de inmediato con un estado que indica que aún no está disponible
- **AND** no intenta calcular recomendaciones en ese momento

### Requirement: Actualización ante nuevas interacciones

El motor DEBE actualizar sus estructuras de recomendación cuando recibe nuevas interacciones, sin afectar la latencia de las consultas en curso.

#### Scenario: Nueva calificación registrada
- **WHEN** el Worker persiste una nueva interacción
- **THEN** el motor programa una actualización en segundo plano
- **AND** las consultas siguen respondiendo con la versión vigente mientras tanto

### Requirement: Indexación inicial tras la curación

El motor DEBE construir su índice o modelo inicial cuando recibe la señal de que el catálogo terminó de curarse.

#### Scenario: Señal de catálogo curado
- **WHEN** el motor recibe el evento de datos listos
- **THEN** inicia la indexación o entrenamiento inicial con el dataset saneado
- **AND** al finalizar informa que está operativo

### Requirement: Disponibilidad condicionada

Las recomendaciones DEBEN permanecer deshabilitadas hasta que la curación del catálogo y la indexación del motor hayan finalizado.

#### Scenario: Consulta antes de estar listo
- **WHEN** el frontend solicita recomendaciones durante el arranque en frío
- **THEN** el sistema responde que la funcionalidad no está disponible todavía
- **AND** no devuelve resultados parciales calculados sobre datos sucios
