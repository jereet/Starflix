# Spec Delta

## Purpose

Desplegar la plataforma completa con un único comando, usando infraestructura oficial contenerizada, redes aisladas y persistencia de datos entre reinicios.

## ADDED Requirements

### Requirement: Arranque con un único comando

El sistema completo DEBE levantarse con `docker compose up --build` desde un clon limpio del repositorio, sin migraciones, configuraciones de colas ni llamadas manuales de inicialización.

#### Scenario: Primer arranque desde un clon limpio
- **WHEN** un evaluador clona el repositorio y ejecuta `docker compose up --build`
- **THEN** todos los contenedores necesarios se construyen y levantan sin intervención manual
- **AND** el frontend queda accesible por HTTP al finalizar la inicialización

#### Scenario: Falla de un servicio dependiente
- **WHEN** MongoDB o RabbitMQ todavía no aceptan conexiones durante el arranque
- **THEN** los servicios dependientes reintentan conectarse en lugar de terminar definitivamente

### Requirement: Infraestructura oficial contenerizada

MongoDB y RabbitMQ DEBEN ejecutarse como contenedores definidos en `docker-compose.yml` usando sus imágenes oficiales de Docker Hub. NO DEBE utilizarse ninguna instancia externa ni instalada fuera del compose.

#### Scenario: Verificación de imágenes oficiales
- **WHEN** se inspecciona `docker-compose.yml`
- **THEN** las definiciones de MongoDB y RabbitMQ referencian imágenes oficiales
- **AND** no existe configuración que apunte a un host externo para esos servicios

### Requirement: Persistencia mediante volúmenes

Los datos de MongoDB DEBEN residir en un volumen de Docker para sobrevivir a la destrucción y recreación de los contenedores.

#### Scenario: Datos conservados tras reinicio
- **WHEN** se ejecuta `docker compose down` sin eliminar volúmenes y luego `docker compose up`
- **THEN** las películas, reseñas y registros de auditoría previos siguen disponibles

#### Scenario: Reinicio desde cero
- **WHEN** se ejecuta `docker compose down -v` y luego `docker compose up --build`
- **THEN** el sistema se comporta como un arranque en frío e inicializa el dataset semilla

### Requirement: Aislamiento de red del frontend

El contenedor del frontend NO DEBE tener conectividad de red con MongoDB ni poder resolver su nombre de host. Todo acceso a datos DEBE estar mediado por el API Gateway y los servicios backend.

#### Scenario: Intento de resolución directa desde el frontend
- **WHEN** se ejecuta una consulta o resolución de nombre hacia el host de MongoDB desde el contenedor del frontend
- **THEN** la operación falla por no existir conectividad ni resolución de nombre
- **AND** las funcionalidades del frontend siguen operativas a través del API Gateway

### Requirement: Inicialización automática del dataset

El contenedor de MongoDB DEBE cargar el dataset semilla provisto por la cátedra mediante scripts en `/docker-entrypoint-initdb.d/`, ejecutándolos solo cuando el volumen de datos está vacío.

#### Scenario: Carga inicial del dataset
- **WHEN** MongoDB arranca sobre un volumen vacío
- **THEN** las colecciones de películas, reseñas y usuarios quedan pobladas con el dataset semilla
- **AND** los scripts de inicialización no vuelven a ejecutarse en arranques posteriores

### Requirement: Configuración por variables de entorno

La configuración de conexiones, credenciales y parámetros de los servicios DEBE provenir de variables de entorno o archivos de configuración del compose, sin credenciales productivas embebidas en el código.

#### Scenario: Cambio de configuración sin recompilar
- **WHEN** se modifican las variables de entorno de un servicio en `docker-compose.yml`
- **THEN** el servicio adopta la nueva configuración al recrearse el contenedor
- **AND** no es necesario modificar el código fuente

### Requirement: Verificación de salud de los servicios

Cada servicio DEBE exponer un mecanismo de verificación de salud que permita al compose conocer cuándo está listo para recibir tráfico.

#### Scenario: Servicio no listo
- **WHEN** un servicio aún no completó su inicialización
- **THEN** su estado de salud se reporta como no saludable
- **AND** el resto del sistema puede distinguir ese estado del de un servicio operativo
