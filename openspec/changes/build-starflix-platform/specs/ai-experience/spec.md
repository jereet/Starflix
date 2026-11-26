# Spec Delta

## Purpose

Agregar valor a la experiencia mediante un modelo de lenguaje que traduzca contenidos del catálogo y sostenga un asistente cinéfilo conversacional basado en los datos reales del sistema.

## ADDED Requirements

### Requirement: Traducción dinámica de contenidos

El sistema DEBE permitir traducir al español, o al idioma seleccionado, las sinopsis y reseñas recuperadas de la base de datos.

#### Scenario: Traducción de una sinopsis
- **WHEN** el usuario solicita ver la sinopsis de una película en español
- **THEN** el sistema devuelve el texto traducido
- **AND** conserva los nombres propios y el título original de la película

#### Scenario: Traducción de reseñas de la comunidad
- **WHEN** el usuario solicita traducir las reseñas de una película
- **THEN** el sistema devuelve los comentarios en el idioma elegido
- **AND** mantiene la calificación y los datos del autor sin alterarlos

### Requirement: Selección de idioma

El frontend DEBE permitir elegir el idioma de los contenidos traducidos entre una suite de idiomas soportados.

#### Scenario: Cambio de idioma
- **WHEN** el usuario selecciona otro idioma disponible
- **THEN** las traducciones posteriores se generan en ese idioma
- **AND** los textos que no requieran traducción permanecen sin cambios

### Requirement: Reutilización de traducciones

El sistema DEBE almacenar o cachear las traducciones generadas para evitar llamadas repetidas al LLM sobre el mismo contenido y el mismo idioma.

#### Scenario: Segunda solicitud del mismo texto
- **WHEN** se solicita una traducción ya generada previamente para el mismo texto e idioma
- **THEN** el sistema devuelve la traducción almacenada
- **AND** no vuelve a invocar al proveedor de LLM

### Requirement: Asistente cinéfilo conversacional

El frontend DEBE ofrecer un chat donde el usuario pueda dialogar con un asistente que conoce el catálogo y justifica sus sugerencias.

#### Scenario: Consulta con criterios de búsqueda
- **WHEN** el usuario pide una recomendación con criterios como género, época o características de la trama
- **THEN** el asistente responde con películas del catálogo que cumplen esos criterios
- **AND** explica brevemente por qué las sugiere

#### Scenario: Consulta comparativa
- **WHEN** el usuario pregunta qué tienen en común dos películas del catálogo
- **THEN** el asistente responde usando la información disponible de ambas películas

### Requirement: Respuestas fundamentadas en el catálogo

El asistente NO DEBE inventar películas ni atributos que no existan en el catálogo. Cuando no encuentre una coincidencia, DEBE indicarlo explícitamente.

#### Scenario: Película fuera del catálogo
- **WHEN** el usuario consulta por una película que no existe en la base
- **THEN** el asistente informa que no está en el catálogo
- **AND** no presenta datos inventados como si fueran reales

### Requirement: Degradación ante fallas del proveedor de LLM

Si el proveedor de LLM no está disponible, las funciones de traducción y chat DEBEN fallar de forma controlada, informando al usuario y sin afectar el resto de la plataforma.

#### Scenario: LLM no disponible durante una traducción
- **WHEN** el usuario solicita una traducción y el proveedor de LLM no responde
- **THEN** el sistema muestra un mensaje de indisponibilidad temporal
- **AND** el catálogo, las reseñas y las recomendaciones siguen funcionando

#### Scenario: LLM no disponible durante el chat
- **WHEN** el usuario envía un mensaje al asistente y el proveedor de LLM no responde
- **THEN** el sistema informa que el asistente no está disponible en ese momento
- **AND** no deja la conversación bloqueada
