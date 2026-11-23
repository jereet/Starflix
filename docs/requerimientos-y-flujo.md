# Requerimientos y flujo

## Flujo de componentes a partir de requerimientos

### RF-01. Exploración del catálogo
Frontend → API Gateway → Catalog Service → Repository → MongoDB(Movies)

Referencia:
* Vista de películas con posters visuales.

### RF-02. Búsqueda y filtrado
Frontend → API Gateway → Catalog Service → Repository → MongoDB(Movies)

Referencia:
* Búsqueda por título y filtros por género

### RF-03. Consulta de información de películas
Frontend → API Gateway → Catalog Service → Repository → MongoDB(Movies) 
→ Repository → MongoDB(Reviews)

Referencia:
* Visualización completa de la información técnica, sinopsis (plot / fullplot), reparto, director y listado navegable de las críticas y calificaciones históricas de la comunidad presentes en la base.

### RF-04. Identificación de usuarios
Frontend → API Gateway → Recommendation Service → Repository → MongoDB (Reviews)

Referencia:
* Un selector simple (dropdown o buscador) que permita elegir con qué usuario del catálogo (usr_0001 a usr_2000) se está interactuando. Esto permite:
  * Ver las recomendaciones personalizadas generadas para ese perfil en base a su historial previo.
  * Emitir nuevos votos en su nombre.

### RF-05. Emisión de valoraciones y reseñas
Frontend → API Gateway → Interaction Service → RabbitMQ → Interaction Worker → Repository → MongoDB(Reviews)

Referencia:
* El usuario califica (escala de 1 a 5 estrellas) y escribe su reseña en una misma acción.

### RF-06. Gestión asincrónica de interacciones
Frontend → API Gateway → Interaction Service → RabbitMQ  
Interaction Worker → RabbitMQ  
       → Repository → MongoDB(Reviews)

Referencia:
* Publicación de una interacción.
* Flujo asincrónico: el frontend envía una interacción que se encola en RabbitMQ. Un microservicio consumidor desacoplado toma el mensaje de la cola y lo persiste de forma segura en MongoDB.

### RF-07. Recomendaciones personalizadas
Frontend → API Gateway → Recommendation Service → [Modelos e índices precalculados] Repository → MongoDB(Movies/Reviews)

Referencia:
* En la página principal: "Recomendadas para vos" (personalizadas según el usuario activo).

### RF-08. Recomendación de películas similares
Frontend → API Gateway → Recommendation Service → [Modelos e índices precalculados] Repository → MongoDB(Movies/Reviews)

Referencia:
* En la ficha de película: "Películas similares" (basadas en afinidad temática o covisualización).

---

Nota del enunciado:  
[IMPORTANTE] El punto neurálgico de creatividad del proyecto: La cátedra otorga libertad técnica absoluta para la concepción del motor de recomendaciones. Pueden utilizar cualquiera de los enfoques vistos en clase o técnicas avanzadas de la industria: 
* Filtrado colaborativo: descomposición matricial o vecinos cercanos con scikit-surprise (SVD, SVD++, NMF, KNNWithMeans) aprovechando la densidad del catálogo de reviews.
* Filtrado basado en contenido: normalización de metadatos (metadata soup con director, reparto, géneros y sinopsis) y similitud coseno mediante bolsa de palabras (CountVectorizer), TF-IDF o embeddings vectoriales. 
* Modelos híbridos y baselines: fórmulas de rating ponderado (ej. baseline de IMDb), esquemas de dos etapas (retrieval + re-ranking).
* Modelos basados en grafos o redes neuronales 

---

### RF-09. Curación y normalización de datos
Cleaning Service → [Reglas de Limpieza] Clean LLM → Repository MCP → Repository → MongoDB(reviews)

Referencia:
* En un principio, los datos del dataset se encuentran con problemas, en el primer levantamiento del servidor deben curarse.

---

Nota del enunciado:  
Reglas determinísticas y validación de integridad:
* Detectar y deduplicar películas redundantes.
* Normalizar tipos corruptos (años como strings "1994" o "2015è", duraciones como "120 min" o negativas).
* Homogeneizar géneros (convertir strings delimitados "Action, Drama" en list[str]).
* Normalizar ratings de IMDb y limpiar espacios espurios en títulos.
* Purgar o clasificar comentarios spam/basura ("...", "", "none", "test").
* Corregir fechas guardadas como strings dispares a tipo fecha ISO (ISODate).
* Eliminar reseñas huérfanas cuyos movie_id no existan en la colección de películas.
* Resolver colisiones de votos duplicados de un mismo usuario sobre una misma película.

---

### RF-10. Resolución semántica mediante LLM
Cleaning Service → Clean LLM → Repository MCP → Repository → MongoDB(reviews)

Referencia:
* Esto es cuando por ejemplo hay un rating null pero se tiene una reseña de ese usuario. En tal caso el LLM debe inferir una puntuación de rating en base a la reseña.

### RF-11. Auditoría de curación de datos
Cleaning Service → Repository → MongoDB (Audit Logs)

Referencia:
* Obtención de logs de auditoría generados por el Clean LLM durante el proceso de curación de datos.

### RF-12. Visualización de auditoría
Frontend → API Gateway → Cleaning Service → Repository → MongoDB (Audit Logs)

Referencia:
* Una vista especializada que permita a los evaluadores inspeccionar el resumen macro de la curación y visualizar el detalle de los registros depurados o inferidos por el agente.

### RF-13. Gestión del estado de inicialización y RF-14. Disponibilidad progresiva de funcionalidades
Cleaning Service → RabbitMQ  
Recommendation service → RabbitMQ

Referencia:
* Hemos determinado que se utilice RabbitMQ para que cleaning service registre la finalización del proceso de limpieza, para que el recommendation service pueda generar las recomendaciones finales.
* Una vez finalizado el proceso de limpieza, recommendation service genera las recomendaciones personalizadas sobre el dataset completo y curado.
* Una vez finalizada la generación del modelo de recomendaciones, se registra en RabbitMQ que el sistema está en su estado final y los próximos pedidos a la api gateway se hacen con funcionalidades completas. API Gateway tiene que guardar de alguna forma dentro de sí mismo este estado para los pedidos próximos, dado que se eliminan de rabbitmq

---

Nota del enunciado:  
[IMPORTANTE] Se deben restringir las funcionalidades dependientes del saneamiento completo: el módulo de recomendaciones debe permanecer deshabilitado o en espera, y la emisión de nuevas valoraciones/reseñas debe pausarse o condicionarse hasta que el catálogo adquiera consistencia.

---

### RF-15. Traducción de contenidos (deseable) y RF-16. Asistente cinéfilo conversacional (deseable)
Frontend → API Gateway → Catalog Service → AI Service → LLM  
                                          Catalog Service → Repository → MongoDB(Movies)

Referencia:
* Traducción dinámica multilingüe de contenidos:  
Dado que el catálogo semilla cuenta con sinopsis y opiniones de usuarios en inglés, es altamente deseable que el sistema integre un LLM para traducir al español (o a un idioma seleccionado por el usuario desde una suite de idiomas) todo texto recuperado de la base de datos (descripciones, tramas y reseñas de la comunidad), preservando preferentemente los nombres propios o títulos originales de las películas.
* Asistente cinéfilo conversacional (chat interactivo):  
Un chat integrado en la interfaz web donde el usuario pueda dialogar interactivamente con una IA cinéfila ("Recomiéndame una película de ciencia ficción de los años 90 con un giro inesperado", "¿Qué tienen en común Inception e Interstellar?"), respondiendo con conocimiento del catálogo y justificando las sugerencias.

---

Propuesta a revisar con el equipo: puede añadirse un AI service repository MCP (revisar nombre) que permita persistir las traducciones en MongoDB, para evitar traducciones continuas sobre contenidos ya vistos previamente.
