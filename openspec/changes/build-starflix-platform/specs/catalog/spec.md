# Spec Delta

## Purpose

Permitir que cualquier visitante explore, busque, filtre y consulte el catálogo cinematográfico y las reseñas históricas de la comunidad, incluso mientras la curación de datos está en curso.

## ADDED Requirements

### Requirement: Exploración paginada del catálogo

El sistema DEBE exponer una vista paginada de películas con poster, título y metadatos básicos para la navegación del catálogo.

#### Scenario: Primera página del catálogo
- **WHEN** el frontend solicita la primera página del catálogo
- **THEN** el sistema devuelve una lista acotada de películas con su poster y datos básicos
- **AND** informa si existen más páginas

#### Scenario: Página sin resultados
- **WHEN** se solicita una página posterior al último elemento disponible
- **THEN** el sistema devuelve una lista vacía sin error

### Requirement: Búsqueda por título

El sistema DEBE permitir buscar películas por título con coincidencia parcial y sin distinguir mayúsculas de minúsculas.

#### Scenario: Coincidencia parcial
- **WHEN** el usuario busca un fragmento del título de una película existente
- **THEN** el sistema devuelve las películas cuyos títulos contienen ese fragmento
- **AND** ordena los resultados de forma estable y comprensible

#### Scenario: Búsqueda sin coincidencias
- **WHEN** el usuario busca un texto que no coincide con ningún título
- **THEN** el sistema devuelve una lista vacía e indica que no hubo resultados

### Requirement: Filtrado por género

El sistema DEBE permitir filtrar el catálogo por uno o más géneros normalizados.

#### Scenario: Filtro por un género
- **WHEN** el usuario selecciona el género Drama
- **THEN** el sistema devuelve únicamente películas que incluyen Drama entre sus géneros

#### Scenario: Combinación de filtro y búsqueda
- **WHEN** el usuario aplica un género y un texto de búsqueda a la vez
- **THEN** el sistema devuelve solo las películas que cumplen ambas condiciones

### Requirement: Detalle completo de una película

El sistema DEBE exponer la ficha de una película con sinopsis corta y extendida, reparto, director, géneros, año, duración y calificación.

#### Scenario: Consulta de una película existente
- **WHEN** el usuario abre la ficha de una película
- **THEN** el sistema devuelve todos los campos disponibles de la película
- **AND** incluye `plot` y `fullplot` cuando existan

#### Scenario: Película inexistente
- **WHEN** el usuario solicita una película cuyo identificador no existe
- **THEN** el sistema responde con un error de recurso no encontrado

### Requirement: Reseñas históricas por película

El sistema DEBE listar las reseñas y calificaciones históricas asociadas a una película, de forma navegable y ordenada.

#### Scenario: Película con reseñas
- **WHEN** el usuario consulta las reseñas de una película que tiene interacciones registradas
- **THEN** el sistema devuelve las reseñas con usuario, calificación, comentario y fecha
- **AND** las presenta en orden cronológico descendente y paginadas

#### Scenario: Película sin reseñas
- **WHEN** el usuario consulta las reseñas de una película sin interacciones
- **THEN** el sistema devuelve una lista vacía sin error

### Requirement: Catálogo legible durante la curación

Durante el arranque en frío, el catálogo DEBE seguir disponible en modo lectura, mostrando posters, descripciones y reseñas existentes aunque los datos aún no estén saneados.

#### Scenario: Exploración durante la curación
- **WHEN** el usuario navega el catálogo mientras el agente de limpieza está trabajando
- **THEN** puede ver películas, descripciones y reseñas previas
- **AND** el sistema informa que la curación está en curso

### Requirement: Consistencia de los datos devueltos

El sistema DEBE devolver los campos del catálogo con tipos estables y predecibles: años y duraciones numéricos, géneros como lista y fechas en formato ISO cuando corresponda.

#### Scenario: Película con datos normalizados
- **WHEN** se consulta una película ya saneada
- **THEN** el año y la duración se devuelven como números
- **AND** los géneros se devuelven como lista de cadenas
- **AND** las fechas se devuelven en formato ISO
