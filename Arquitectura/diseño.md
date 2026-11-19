# Relaciones a partir de Requisitos Funcionales 

## **RF-01. Exploración del catálogo**

Frontend -> API Gateway -> Catalog Service -> MongoDB(Movies)

## **RF-02. Búsqueda y filtrado**

Frontend -> API Gateway -> Catalog Service -> MongoDB(Movies)

## **RF-03. Consulta de información de películas**

Frontend -> API Gateway -> Catalog Service -> MongoDB(Movies)

## **RF-04. Identificación de usuarios**

Frontend -> API Gateway -> Catalog Service -> MongoDB (Users)

## **RF-05. Emisión de valoraciones y reseñas**

Frontend -> API Gateway -> Interaction Service -> RabbitMQ -> Interaction Worker -> MongoDB(Reviews)

## **RF-06. Gestión asincrónica de interacciones**

Frontend -> API Gateway -> Interaction Service -> RabbitMQ -> Interaction Worker -> MongoDB(Reviews)

## **RF-07. Recomendaciones personalizadas**

Frontend -> API Gateway -> Recommendation Service -> [Modelos e indices precalculados] MongoDB(Movies/Reviews)

## **RF-08. Recomendación de películas similares**

Frontend -> API Gateway -> Recommendation Service -> [Modelos e indices precalculados] MongoDB(Movies/Reviews)

## **RF-09. Curación y normalización de datos**

MongoDB -> Cleaning Service -> [Reglas de Limpieza] Clean LLM -> MongoDB(Movies)


## **RF-10. Resolución semántica mediante LLM**

MongoDB -> Cleaning Service -> Clean LLM -> [Resultado inferido] Cleaning Service -> MongoDB(Movies)


## **RF-11. Auditoría de curación de datos**

Cleaning Service -> MongoDB (Audit Logs)

Podría ser un módulo del propio Cleaning Service encargado de registrar:
- Tipo de anomalía.
- Registro afectado.
- Valor original.
- Valor corregido.
- Motivo de la corrección.
- Resultado de la inferencia del LLM, cuando corresponda.

## **RF-12. Visualización de auditoría**

Frontend -> API Gateway -> Cleaning Service -> MongoDB (Audit Logs)

Cleaning Service puede ofrecer endpoints específicos para consultar las métricas generales y los detalles de cada modificación.

## **RF-13. Gestión del estado de inicialización** y **RF-14. Disponibilidad progresiva de funcionalidades**

Cleaning Service -> System State <-> [Recommendation service] /
                                 <-> API Gateway -> Frontend

El estado debería contemplar, como mínimo, dos procesos:
- Estado de curación del catálogo.
- Estado de inicialización del recomendador.

El frontend consulta el estado a través del Gateway y habilita o restringe las funcionalidades correspondientes.

## **RF-15. Traducción de contenidos (deseable)** y **RF-16. Asistente cinéfilo conversacional (deseable)**

Frontend -> API Gateway -> AI Service -> LLM /
                                      -> Catalog Service -> MongoDB(Movies)


# Componentes identificados

## Frontend (Dominio: Interfaz de usuario) - Microservicio

Frontend es el encargado de comunicarse con el usuario y mostrar la información del sistema.

## API Gateway (Dominio: Entrada y distribucion de solicitudes) - Microservicio

API Gateway es la unica entrada al backend por medio del frontend, recibe las solicitudes y las dirige al microservicio correspondiente.

## Catalog Service (Dominio: Catalogo cinematografico) - Microservicio

Sera el servicio encargado de:
* Obtener las películas.
* Buscar por título.
* Filtrar por género.
* Obtener los detalles de una película.
* Obtener sus reseñas.

## Interaction Service  - Microservicio & Interaction Worker - Proceso (Dominio: Valoraciones, reseñas y procesamiento asincrónico)

Interaction Service se encarga de recibir y publicar el mensaje.

Interaction Worker se encarga de consumirlo y guardarlo.

## Recommendation service (Dominio: Recomendaciones) - Microservicio

* Leer películas, valoraciones y reseñas de MongoDB.
* Construir los modelos de recomendación.
* Generar y mantener índices o resultados precalculados (No debería consultar y procesar todo el catálogo desde cero cada vez que recibe una solicitud).
* Responder solicitudes de recomendaciones personalizadas.
* Obtener películas similares.
* Actualizar sus modelos en cuanto recibe nuevas interacciones o cuando se cure el catalogo.

## Cleaning Service: (Dominio: Curación de datos) - Microservicio

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

## System State (Dominio: Estado del sistema) - Componente

System State se encarga de guardar y mostrar el estado del sistema, un cambio en ella hace que haya un cambio directo en el frontend (¿Subscriber?).

## IA Service: (Dominio: IA) - Microservicio

El servicio se encarga de:
* Traducción dinamica de contenido
o
* Chat interactivo 
