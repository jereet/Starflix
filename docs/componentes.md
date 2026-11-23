# Componentes Arquitectónicos

## Frontend - Cliente
Frontend es el encargado de comunicarse con el usuario y mostrar la información del sistema.

## API Gateway - Microservicio
API Gateway es la única entrada al backend por medio del frontend, recibe las solicitudes y las dirige al microservicio correspondiente.

## Catalog Service - Microservicio
Será el servicio encargado de:
* Obtener las películas.
* Buscar por título.
* Filtrar por género.
* Obtener los detalles de una película.
* Obtener sus reseñas.

## Interaction Service e Interaction Worker - Microservicios
Interaction Service funciona como servicio intermediario para la publicación de reseñas y valoraciones. Encola en RabbitMQ para el procesamiento con 0 delay.  
Interaction Worker las procesa a medida que dispone de tiempo.

## Recommendation service - Microservicio
Funciones:
* Leer valoraciones y reseñas de MongoDB.
* Construir los modelos de recomendación.
* Implementar algún cacheado de recomendaciones (no definitivo, evaluar).
* Responder solicitudes de recomendaciones personalizadas.
* Obtener películas similares.
* Actualizar sus modelos en cuanto recibe nuevas interacciones o cuando se cure el catálogo (evaluar si es necesaria la actualización de modelos).

Servicios concretos desde una perspectiva funcional:
* recomendaciones generales por puntuación (ponderada en clase collab)
   * primera vez puede ser ordenada por rating, limitada a pocas reviews
* recomendaciones de películas similares al entrar a una peli
* recomendaciones de películas que pueden gustar en base a las miradas (revieweadas por el usuario)

## Cleaning Service - Microservicio
1. Detecta una reseña con problemas.
* Duplicados.
* Años y duraciones inválidos.
* Géneros incorrectos.
* Ratings de IMDb anómalos.
* Espacios innecesarios en títulos.
* Reviews spam.
* Fechas inválidas.
* Reseñas huérfanas.
* Votos duplicados.
2. Determina que necesita una interpretación semántica.
3. Envía el comentario al LLM.
4. Recibe una calificación inferida.
5. Valida el resultado.
6. Actualiza MongoDB y registra la operación en la auditoría.

## AI Service - Microservicio
El servicio se encarga de la traducción dinámica de contenido. Podría incluir:
* Chat interactivo (Evaluar).
* Persistencia de traducciones en BD (a definir)
