# Notas y propuestas

## Creación de usuarios
Ahora mismo por enunciado se determina que hay 2000 usuarios ya creados en la base de datos review. Estos usuarios son estáticos y no tienen ningún mecanismo de autenticación asociados.

El equipo sugiere:
* Creación de usuarios dinamica
* Que los usuarios ya registrados en el dataset se carguen a una nueva base de datos con sus pertinentes reviews en la cúal se puedan eliminar/modificar  

Esto lo podemos implementar mediante un microservicio adicional.
