# ACI606 - Docker Rescue Challenge

## 1. Descripción del proyecto

Este proyecto consiste en recuperar y mejorar una aplicación web
compuesta por tres servicios que funcionan mediante Docker Compose.

El trabajo se realizó sobre el proyecto original entregado para la
evaluación, conservando su arquitectura y sus componentes.

## 2. Arquitectura

La aplicación está compuesta por:

- Frontend: Nginx.
- Backend: FastAPI con Python.
- Base de datos: PostgreSQL.

Comunicación entre servicios:

Frontend (Nginx) -> Backend (FastAPI) -> PostgreSQL.

## 3. Requisitos

Para ejecutar el proyecto se necesita:

- Docker instalado.
- Docker Compose.
- Git.
- Acceso a una terminal.

## 4. Configuración inicial

Primero, clonar el repositorio:

    git clone https://github.com/jhoxwe-lgtm/aci606-docker-rescue.git

Entrar en la carpeta:

    cd aci606-docker-rescue

Crear el archivo de configuración:

    cp .env.example .env

Abrir el archivo .env y configurar una contraseña propia.

Las variables utilizadas son:

- POSTGRES_USER
- POSTGRES_PASSWORD
- POSTGRES_DB
- APP_PORT

Importante: el archivo .env contiene información privada y no debe
subirse al repositorio. Está protegido mediante .gitignore.

## 5. Iniciar la aplicación

Ejecutar:

    docker compose up -d --build

Este comando construye las imágenes e inicia los tres servicios.

## 6. Comprobar el funcionamiento

Consultar el estado de los servicios:

    docker compose ps

Los servicios deben alcanzar el estado healthy:

- frontend
- backend
- db

Revisar los registros:

    docker compose logs

Comprobar la salud de la API:

    curl http://localhost:8080/api/health

Respuesta esperada:

    {"status":"ok","database":"reachable"}

La aplicación se encuentra disponible en:

    http://localhost:8080

En GitHub Codespaces también se puede abrir desde la pestaña
Puertos, seleccionando el puerto 8080.

## 7. Operaciones de la API

Consultar los registros:

    curl http://localhost:8080/api/items

Crear un registro de prueba:

    curl -X POST http://localhost:8080/api/items \
      -H "Content-Type: application/json" \
      -d '{"name":"Prueba de persistencia Docker"}'

La API utiliza las siguientes rutas:

- GET /api/health
- GET /api/items
- POST /api/items

## 8. Persistencia de datos

PostgreSQL utiliza un volumen nombrado:

    postgres_data

Este volumen conserva los datos aunque se recree el contenedor
de la base de datos.

Para comprobar la persistencia:

1. Crear un registro mediante la API.
2. Consultar los registros.
3. Recrear el contenedor de PostgreSQL:

       docker compose up -d --force-recreate db

4. Esperar a que PostgreSQL esté saludable.
5. Consultar nuevamente los registros.

El registro creado debe seguir existiendo.

Importante: docker compose down no elimina el volumen.
No utilizar docker compose down -v, porque elimina los volúmenes
y puede borrar los datos almacenados.

## 9. Prueba de fallo controlado y recuperación

Para simular una falla del backend:

    docker compose stop backend

Consultar el estado:

    docker compose ps -a

Comprobar la API:

    curl -i http://localhost:8080/api/health

Durante la prueba se obtiene una respuesta HTTP 502 Bad Gateway,
porque Nginx no puede comunicarse con el backend detenido.

Para recuperar el servicio:

    docker compose start backend

Comprobar nuevamente:

    docker compose ps

    curl http://localhost:8080/api/health

La aplicación debe volver a responder correctamente y conservar
los registros almacenados en PostgreSQL.

## 10. Incidencias identificadas y reparaciones

R-01: Se corrigió el módulo utilizado para iniciar Uvicorn.
Se cambió app:app por main:app.

R-02: Se agregó la dependencia psycopg2-binary para permitir
la conexión entre SQLAlchemy y PostgreSQL.

R-03: Se corrigió la dirección de conexión a PostgreSQL.
Se reemplazó localhost por el nombre del servicio db.

R-04: Se corrigió la contraseña utilizada por el backend para
que coincidiera con la configuración de PostgreSQL.

R-05: Se corrigió la dirección de comunicación de Nginx.
Se reemplazó localhost por backend.

R-06: Se corrigió la publicación del puerto del frontend.
Se configuró 8080:80, respetando el puerto interno de Nginx.

R-07: Se incorporó un volumen nombrado para conservar los datos
de PostgreSQL y se realizó una prueba de persistencia.

R-08: Se incorporaron comprobaciones de salud para los tres
servicios mediante healthchecks.

R-09: Se realizó una prueba de fallo controlado del backend
y posteriormente se comprobó su recuperación.

## 11. Evidencias

La carpeta evidencias contiene los registros utilizados para
documentar los diagnósticos, las reparaciones y las pruebas.

Incluye evidencias de:

- Errores originales del backend.
- Problemas de conexión con PostgreSQL.
- Configuración y comunicación de Nginx.
- Funcionamiento de la API.
- Persistencia de datos.
- Estado de salud de los servicios.
- Fallo controlado y recuperación.

## 12. Detener la aplicación

Para detener los contenedores:

    docker compose down

Este comando detiene y elimina los contenedores y la red,
pero conserva el volumen nombrado de PostgreSQL.

## 13. Resultado final

Se logró recuperar la aplicación original y corregir las
incidencias identificadas.

Se comprobó el funcionamiento de los tres servicios, la
comunicación entre ellos, la persistencia de los datos,
los healthchecks y la recuperación después de un fallo.

El proyecto conserva el historial de modificaciones mediante
commits publicados en GitHub.
