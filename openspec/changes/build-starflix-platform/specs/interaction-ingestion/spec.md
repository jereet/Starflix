# Spec Delta

## Purpose

Recibir votos y reseñas en una sola acción y persistirlos de forma asincrónica a través de RabbitMQ, absorbiendo ráfagas sin bloquear al usuario ni escribir directamente en MongoDB desde el cliente.

## ADDED Requirements

### Requirement: Emisión unificada de voto y reseña

El frontend DEBE permitir calificar una película de 1 a 5 estrellas y escribir un comentario en una misma acción.

#### Scenario: Envío de una interacción completa
- **WHEN** el usuario elige una calificación y escribe un comentario
- **THEN** el sistema recibe ambos datos como una única interacción
- **AND** confirma la recepción al usuario sin esperar la persistencia definitiva

#### Scenario: Comentario opcional
- **WHEN** el usuario envía solo una calificación sin comentario
- **THEN** el sistema acepta la interacción y registra la reseña sin texto

### Requirement: Publicación asincrónica en RabbitMQ

El Interaction Service DEBE publicar cada interacción válida como un mensaje en RabbitMQ y responder al cliente sin escribir directamente en MongoDB.

#### Scenario: Ráfaga de interacciones
- **WHEN** llegan muchas interacciones en un período corto
- **THEN** el servicio las publica en la cola y responde a cada solicitud de forma rápida
- **AND** el frontend no queda bloqueado esperando la escritura en la base

#### Scenario: Confirmación de encolado
- **WHEN** una interacción válida se publica correctamente
- **THEN** el sistema responde con una confirmación de aceptación
- **AND** la interacción todavía no aparece necesariamente en el catálogo

### Requirement: Persistencia desacoplada por el Interaction Worker

Un proceso consumidor independiente DEBE tomar los mensajes de la cola y persistirlos de forma segura en la colección de reseñas.

#### Scenario: Consumo normal de la cola
- **WHEN** el Interaction Worker recibe un mensaje de interacción
- **THEN** lo valida y lo persiste en MongoDB
- **AND** confirma el procesamiento para que el mensaje no se reprocese

#### Scenario: Interacción persistida visible
- **WHEN** el Worker terminó de persistir una interacción
- **THEN** la nueva reseña aparece en la ficha de la película y en el historial del usuario

### Requirement: Idempotencia y resolución de duplicados

El sistema DEBE evitar que un mismo usuario acumule votos duplicados sobre la misma película, resolviendo la colisión de forma determinista.

#### Scenario: Segundo voto del mismo usuario
- **WHEN** un usuario vuelve a calificar una película que ya calificó
- **THEN** el sistema actualiza su voto existente en lugar de crear un registro duplicado
- **AND** el historial muestra una única calificación vigente por usuario y película

#### Scenario: Reintento de un mensaje ya procesado
- **WHEN** el Worker recibe nuevamente un mensaje que ya había persistido
- **THEN** el resultado final no cambia y no se generan duplicados

### Requirement: Validación de interacciones

El sistema DEBE rechazar interacciones con calificaciones fuera del rango 1 a 5, usuarios inexistentes o películas inexistentes, sin publicarlas en la cola.

#### Scenario: Calificación inválida
- **WHEN** llega una interacción con calificación 0 o 6
- **THEN** el sistema responde con un error de solicitud inválida
- **AND** no se publica ningún mensaje

#### Scenario: Película inexistente
- **WHEN** llega una interacción para una película que no existe
- **THEN** el sistema la rechaza y no la persiste

### Requirement: Reintentos y mensajes fallidos

Si la persistencia falla por una causa transitoria, el sistema DEBE reintentar el procesamiento y DEBE derivar a una cola de mensajes fallidos los que no puedan procesarse.

#### Scenario: Base de datos temporalmente no disponible
- **WHEN** el Worker no puede escribir en MongoDB
- **THEN** reintenta el mensaje según la política configurada
- **AND** no pierde la interacción mientras los reintentos estén disponibles

#### Scenario: Mensaje irrecuperable
- **WHEN** un mensaje supera la cantidad máxima de reintentos
- **THEN** se mueve a una cola de mensajes fallidos para su inspección
- **AND** el resto de la cola sigue procesándose

### Requirement: Pausa durante la curación

Mientras el catálogo no esté saneado, el sistema DEBE rechazar o pausar la emisión de nuevas interacciones e informar el motivo.

#### Scenario: Intento de voto durante el arranque en frío
- **WHEN** el usuario intenta enviar un voto antes de que la curación finalice
- **THEN** el sistema rechaza la interacción con un estado que indica que aún no está disponible
- **AND** el frontend muestra que la emisión de reseñas está temporalmente deshabilitada

#### Scenario: Reanudación tras la curación
- **WHEN** el sistema confirma que el catálogo está curado
- **THEN** la emisión de nuevas interacciones se habilita automáticamente
