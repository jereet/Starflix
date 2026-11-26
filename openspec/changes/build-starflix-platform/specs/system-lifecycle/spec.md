# Spec Delta

## Purpose

Coordinar el arranque en frío y en caliente del sistema, propagar el estado de curación e indexación y habilitar las funcionalidades del frontend de forma progresiva.

## ADDED Requirements

### Requirement: Detección del tipo de arranque

El sistema DEBE distinguir entre un arranque en frío, con datos crudos o volumen vacío, y un arranque en caliente, con datos ya saneados.

#### Scenario: Volumen vacío
- **WHEN** el sistema arranca sobre una base sin datos curados
- **THEN** se clasifica como arranque en frío e inicia el proceso de curación

#### Scenario: Volumen con datos curados
- **WHEN** el sistema arranca sobre una base cuyo catálogo ya fue saneado
- **THEN** se clasifica como arranque en caliente y omite la limpieza completa

### Requirement: Secuencia de arranque en frío

En un arranque en frío, el sistema DEBE inicializar el dataset, ejecutar la curación y luego disparar la indexación del recomendador antes de habilitar las funcionalidades dependientes.

#### Scenario: Primera ejecución completa
- **WHEN** se ejecuta `docker compose up --build` sobre un entorno limpio
- **THEN** MongoDB carga el dataset semilla
- **AND** el agente de limpieza diagnostica y sanea los datos
- **AND** al finalizar se dispara la indexación del recomendador
- **AND** al completarse, el frontend habilita todas sus capacidades

### Requirement: Arranque en caliente rápido

En un arranque en caliente, los servicios DEBEN detectar el estado saneado previo y quedar operativos en pocos segundos, con el frontend plenamente habilitado desde el inicio.

#### Scenario: Reinicio con datos persistidos
- **WHEN** se reinicia el sistema sobre un volumen ya curado
- **THEN** no se repite la limpieza pesada ni la indexación inicial completa
- **AND** el frontend muestra todas las funcionalidades habilitadas desde el primer momento

### Requirement: Evento de catálogo curado

Al finalizar la curación, el Cleaning Service DEBE publicar en RabbitMQ un evento que indique que los datos están listos.

#### Scenario: Publicación del evento de datos listos
- **WHEN** el agente termina de sanear el catálogo
- **THEN** publica un evento de catálogo curado en RabbitMQ
- **AND** el evento incluye información suficiente para que el recomendador inicie su indexación

### Requirement: Evento de recomendador operativo

Al finalizar su indexación inicial, el Recommendation Service DEBE publicar en RabbitMQ un evento que indique que las recomendaciones están disponibles.

#### Scenario: Publicación del evento de recomendaciones listas
- **WHEN** el recomendador completa la indexación inicial
- **THEN** publica un evento de recomendaciones operativas
- **AND** el API Gateway actualiza el estado del sistema en consecuencia

### Requirement: Estado del sistema consultable

El sistema DEBE exponer el estado de curación del catálogo y de inicialización del recomendador para que el frontend ajuste la interfaz.

#### Scenario: Estado durante la curación
- **WHEN** el frontend consulta el estado mientras la curación está en curso
- **THEN** recibe un estado que indica curación en progreso y recomendaciones no disponibles

#### Scenario: Estado con recomendador indexando
- **WHEN** la curación terminó pero el recomendador aún se está indexando
- **THEN** recibe un estado que indica catálogo curado y recomendaciones en preparación

#### Scenario: Estado plenamente operativo
- **WHEN** ambos procesos finalizaron
- **THEN** recibe un estado que indica que todas las funcionalidades están habilitadas

### Requirement: Disponibilidad progresiva de funcionalidades

El frontend DEBE mantener siempre disponible la exploración del catálogo y la lectura de reseñas, y DEBE restringir recomendaciones y nuevas interacciones hasta que el estado lo permita.

#### Scenario: Funcionalidades durante el arranque en frío
- **WHEN** el sistema está en arranque en frío
- **THEN** el catálogo, los posters, las descripciones y las reseñas previas son visibles
- **AND** las recomendaciones y el envío de nuevas valoraciones están deshabilitados
- **AND** la interfaz informa que la curación está en curso

#### Scenario: Funcionalidades con el sistema listo
- **WHEN** el estado pasa a plenamente operativo
- **THEN** el frontend habilita recomendaciones y emisión de interacciones
- **AND** el cambio se refleja sin requerir una recarga manual de la página

### Requirement: Persistencia del estado entre reinicios

El último estado conocido del sistema DEBE conservarse fuera de la memoria volátil de los mensajes, de modo que un reinicio de servicios no haga retroceder la interfaz a un estado anterior.

#### Scenario: Reinicio del API Gateway
- **WHEN** el API Gateway se reinicia después de alcanzar el estado plenamente operativo
- **THEN** al volver a responder informa el estado operativo sin esperar un nuevo evento de RabbitMQ

#### Scenario: Reinicio durante la curación
- **WHEN** un servicio se reinicia mientras la curación está en curso
- **THEN** el sistema retoma el proceso y mantiene un estado coherente para el frontend
