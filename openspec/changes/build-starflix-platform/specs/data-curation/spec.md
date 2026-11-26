# Spec Delta

## Purpose

Diagnosticar y sanear las anomalías del dataset cinematográfico aplicando reglas determinísticas y, para ambigüedades semánticas, inferencia con un modelo de lenguaje, registrando cada corrección para su auditoría.

## ADDED Requirements

### Requirement: Deduplicación de películas

El agente DEBE detectar películas redundantes y conservar una única versión canónica por película.

#### Scenario: Películas duplicadas
- **WHEN** dos o más documentos representan la misma película
- **THEN** el agente conserva un documento canónico
- **AND** elimina o marca los duplicados
- **AND** registra la decisión en la auditoría

### Requirement: Normalización de años

El agente DEBE convertir los años corruptos o almacenados como texto a un valor numérico válido.

#### Scenario: Año como cadena numérica
- **WHEN** el año de una película está almacenado como `"1994"`
- **THEN** el agente lo convierte al número `1994`

#### Scenario: Año con caracteres espurios
- **WHEN** el año contiene caracteres adicionales, como `"2015è"`
- **THEN** el agente extrae el año válido y lo guarda como número
- **AND** registra el valor original y el corregido

### Requirement: Normalización de duraciones

El agente DEBE convertir las duraciones a un número de minutos positivo, descartando formatos textuales y valores inválidos.

#### Scenario: Duración con unidad
- **WHEN** la duración está almacenada como `"120 min"`
- **THEN** el agente la convierte al número `120`

#### Scenario: Duración negativa
- **WHEN** la duración es negativa o no representa una cantidad válida de minutos
- **THEN** el agente la corrige o la marca como no disponible
- **AND** registra la anomalía en la auditoría

### Requirement: Homogeneización de géneros

El agente DEBE convertir los géneros delimitados por texto en una lista de cadenas normalizadas.

#### Scenario: Géneros como cadena delimitada
- **WHEN** los géneros están almacenados como `"Action, Drama"`
- **THEN** el agente los convierte en la lista `["Action", "Drama"]`
- **AND** elimina espacios y duplicados internos

### Requirement: Normalización de calificaciones y títulos

El agente DEBE normalizar las calificaciones de IMDb al rango esperado y eliminar espacios espurios al inicio y al final de los títulos.

#### Scenario: Rating de IMDb fuera de rango
- **WHEN** una calificación de IMDb es negativa o supera el máximo admitido
- **THEN** el agente la corrige o la marca como no disponible
- **AND** registra la corrección

#### Scenario: Título con espacios espurios
- **WHEN** un título tiene espacios al inicio, al final o repetidos
- **THEN** el agente lo guarda con un único espaciado consistente

### Requirement: Purga o clasificación de comentarios basura

El agente DEBE detectar comentarios vacíos o sin contenido útil, como `"..."`, `""`, `"none"` o `"test"`, y excluirlos de las reseñas válidas.

#### Scenario: Comentario sin contenido
- **WHEN** una reseña contiene solo texto vacío o una cadena de relleno conocida
- **THEN** el agente la elimina o la marca como spam
- **AND** no se contabiliza como una reseña válida del catálogo

### Requirement: Normalización de fechas

El agente DEBE convertir las fechas almacenadas como cadenas dispares a un tipo de fecha ISO consistente.

#### Scenario: Fecha en formato no uniforme
- **WHEN** una fecha de reseña está almacenada como texto en un formato reconocible pero distinto al estándar
- **THEN** el agente la convierte a una fecha ISO
- **AND** registra el valor original y el normalizado

#### Scenario: Fecha no interpretable
- **WHEN** una fecha no puede interpretarse de forma confiable
- **THEN** el agente la marca como inválida y la registra para auditoría
- **AND** no inventa una fecha

### Requirement: Eliminación de reseñas huérfanas

El agente DEBE eliminar las reseñas cuyo `movie_id` no corresponda a ninguna película existente.

#### Scenario: Reseña sin película asociada
- **WHEN** una reseña referencia un `movie_id` inexistente en la colección de películas
- **THEN** el agente la elimina
- **AND** registra la eliminación en la auditoría

### Requirement: Resolución de votos duplicados

El agente DEBE resolver las colisiones de votos de un mismo usuario sobre una misma película, dejando una única calificación vigente.

#### Scenario: Votos repetidos del mismo usuario
- **WHEN** existen varios votos del mismo usuario para la misma película
- **THEN** el agente conserva una única calificación según una regla determinista
- **AND** registra cuántos votos se consolidaron

### Requirement: Inferencia semántica de calificaciones con LLM

El agente DEBE usar un modelo de lenguaje para inferir una calificación entera de 1 a 5 en reseñas cuyo rating es nulo pero cuyo comentario contiene una opinión evaluable.

#### Scenario: Reseña con rating nulo y texto evaluable
- **WHEN** una reseña tiene `rating: null` y un comentario en inglés con contenido de opinión
- **THEN** el agente envía el texto al LLM
- **AND** registra la calificación entera inferida en la reseña
- **AND** guarda la justificación emitida por el modelo en la auditoría

#### Scenario: Reseña nula sin contenido evaluable
- **WHEN** una reseña tiene `rating: null` y un comentario vacío o sin opinión
- **THEN** el agente no inventa una calificación
- **AND** la marca para revisión o la descarta según la regla de spam

### Requirement: Validación de la inferencia del LLM

El agente DEBE validar la respuesta del LLM antes de persistirla, aceptando solo calificaciones enteras dentro del rango 1 a 5.

#### Scenario: Respuesta válida del LLM
- **WHEN** el LLM devuelve una calificación entera entre 1 y 5
- **THEN** el agente la acepta y la persiste

#### Scenario: Respuesta inválida del LLM
- **WHEN** el LLM devuelve un valor fuera de rango, no entero o no interpretable
- **THEN** el agente rechaza el resultado y reintenta o deja la reseña sin calificación
- **AND** registra el intento fallido en la auditoría

### Requirement: Degradación ante fallas del LLM

Si el proveedor de LLM no está disponible, el agente DEBE continuar con las reglas determinísticas y dejar pendientes las inferencias semánticas sin corromper datos.

#### Scenario: LLM no disponible
- **WHEN** el agente no puede comunicarse con el LLM
- **THEN** completa el saneamiento determinístico
- **AND** deja las reseñas pendientes de inferencia identificadas para un intento posterior
- **AND** informa el estado degradado en la auditoría

### Requirement: Registro de auditoría de cada corrección

El agente DEBE registrar cada anomalía detectada y cada corrección aplicada con tipo de anomalía, colección, identificador del registro, valor original, valor corregido, motivo y resultado del LLM cuando corresponda.

#### Scenario: Corrección determinística auditada
- **WHEN** el agente corrige una anomalía
- **THEN** crea un registro de auditoría con el antes y el después
- **AND** incluye el motivo de la corrección

#### Scenario: Inferencia auditada
- **WHEN** el agente infiere una calificación con el LLM
- **THEN** el registro de auditoría incluye la calificación inferida y la justificación del modelo

### Requirement: Idempotencia y reanudación

El agente DEBE detectar que el catálogo ya fue saneado y NO DEBE repetir el proceso pesado de limpieza en arranques posteriores.

#### Scenario: Arranque sobre datos ya curados
- **WHEN** el agente inicia y el catálogo ya está marcado como saneado
- **THEN** omite la limpieza completa
- **AND** confirma el estado curado para permitir el arranque rápido del resto del sistema

#### Scenario: Interrupción durante la limpieza
- **WHEN** el agente se interrumpe antes de finalizar
- **THEN** al reiniciar retoma el trabajo pendiente sin volver a corregir lo ya corregido
