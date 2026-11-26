# Spec Delta

## Purpose

Permitir que el visitante elija una identidad entre los usuarios existentes del catálogo y realice acciones en su nombre, sin un sistema de autenticación.

## ADDED Requirements

### Requirement: Selector de usuario

El frontend DEBE ofrecer un selector que permita elegir uno de los usuarios `usr_0001` a `usr_2000` mediante lista desplegable o búsqueda.

#### Scenario: Listado de usuarios disponibles
- **WHEN** el usuario abre el selector de identidad
- **THEN** el sistema muestra los usuarios disponibles del catálogo
- **AND** permite buscarlos por identificador

#### Scenario: Selección de un usuario
- **WHEN** el usuario elige un identificador válido
- **THEN** el sistema lo establece como identidad activa
- **AND** actualiza las vistas dependientes del perfil, como las recomendaciones personalizadas

### Requirement: Persistencia de la identidad activa

La identidad seleccionada DEBE conservarse en el cliente entre recargas de la página y navegaciones dentro del frontend.

#### Scenario: Recarga de la página
- **WHEN** el usuario recarga el frontend después de elegir una identidad
- **THEN** el sistema mantiene al mismo usuario como identidad activa

#### Scenario: Sin identidad seleccionada
- **WHEN** el usuario accede por primera vez sin identidad previa
- **THEN** el sistema solicita elegir un usuario antes de emitir votos o ver recomendaciones personalizadas

### Requirement: Acciones atribuidas al usuario activo

Toda interacción emitida desde el frontend DEBE registrarse a nombre de la identidad activa.

#### Scenario: Emisión de un voto con identidad activa
- **WHEN** el usuario envía una calificación y reseña con una identidad seleccionada
- **THEN** la interacción se asocia al identificador de esa identidad

### Requirement: Rechazo de identidades inválidas

El sistema DEBE rechazar identificadores que no pertenezcan al conjunto de usuarios existentes.

#### Scenario: Identificador fuera del conjunto
- **WHEN** el frontend o un cliente envía una solicitud con un usuario inexistente
- **THEN** el sistema responde con un error de solicitud inválida
- **AND** no registra ninguna interacción
