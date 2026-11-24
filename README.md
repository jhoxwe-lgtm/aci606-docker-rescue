# ACI606 - Docker Rescue Challenge

## 1. Descripción

Este repositorio contiene la recuperación de la aplicación multi-servicio entregada para la Evaluación 2 de ACI606. Se trabajó sobre el proyecto original, conservando sus tres componentes:

- **Frontend:** Nginx.
- **Backend:** FastAPI.
- **Base de datos:** PostgreSQL.

La comunicación sigue el flujo:

`Navegador → Nginx → FastAPI → PostgreSQL`

## 2. Requisitos

- Docker instalado y en ejecución.
- Docker Compose.
- Git.
- Una terminal.

## 3. Descargar y configurar

Clonar el repositorio:

```bash
git clone https://github.com/jhoxwe-lgtm/aci606-docker-rescue.git
cd aci606-docker-rescue
```

Crear el archivo local de configuración:

```bash
cp .env.example .env
```

Editar `.env` y definir una contraseña propia para `POSTGRES_PASSWORD`. Las variables disponibles son:

- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_DB`
- `APP_PORT`

El archivo `.env` es local y está excluido de Git mediante `.gitignore`. No se debe publicar ni compartir con contraseñas reales. El archivo `.env.example` contiene únicamente valores de ejemplo.

## 4. Construir e iniciar los servicios

Desde la carpeta del proyecto, ejecutar:

```bash
docker compose up --build -d
```

Este comando construye las imágenes e inicia frontend, backend y PostgreSQL.

Consultar el estado:

```bash
docker compose ps
```

Los tres servicios deben llegar al estado `healthy`. Para revisar los registros:

```bash
docker compose logs
```

## 5. Verificar el funcionamiento

Comprobar la salud de la API a través de Nginx:

```bash
curl -i http://localhost:8080/api/health
```

Respuesta esperada:

```json
{"status":"ok","database":"reachable"}
```

Abrir la aplicación en:

`http://localhost:8080`

En GitHub Codespaces también se puede abrir el puerto 8080 desde la pestaña **Puertos**.

## 6. Consultar y crear registros

Consultar los registros guardados:

```bash
curl http://localhost:8080/api/items
```

Crear un registro de prueba:

```bash
curl -X POST http://localhost:8080/api/items \
  -H "Content-Type: application/json" \
  -d '{"name":"Prueba de persistencia Docker"}'
```

Rutas disponibles:

- `GET /api/health`: comprueba el backend y la conexión a la base de datos.
- `GET /api/items`: lista los registros.
- `POST /api/items`: crea un registro.

## 7. Persistencia de PostgreSQL

PostgreSQL utiliza el volumen nombrado `postgres_data`, montado en `/var/lib/postgresql/data`. El volumen permite conservar los datos al recrear el contenedor de la base de datos.

Para comprobarlo:

1. Crear un registro mediante `POST /api/items`.
2. Consultar los registros mediante `GET /api/items`.
3. Recrear únicamente el contenedor de PostgreSQL:

   ```bash
   docker compose up -d --force-recreate db
   ```

4. Esperar a que el servicio vuelva a estar saludable:

   ```bash
   docker compose ps
   ```

5. Consultar nuevamente `/api/items`. El registro creado debe seguir presente.

**Importante:** `docker compose down` conserva el volumen nombrado. No ejecutar `docker compose down -v` durante esta prueba, porque la opción `-v` elimina los volúmenes y puede borrar los datos.

## 8. Fallo controlado y recuperación

Detener el backend:

```bash
docker compose stop backend
docker compose ps -a
```

Comprobar el efecto sobre la API:

```bash
curl -i --max-time 5 http://localhost:8080/api/health
```

Durante la prueba se espera una respuesta `502 Bad Gateway`, porque Nginx no puede comunicarse con el backend detenido.

Recuperar el backend:

```bash
docker compose start backend
docker compose ps
curl -i http://localhost:8080/api/health
```

Después de que el backend vuelva a estar saludable, la API debe responder `HTTP 200 OK` y los registros deben mantenerse.

## 9. Detener y recrear toda la aplicación

Para detener y eliminar los contenedores y la red, conservando el volumen de PostgreSQL:

```bash
docker compose down
```

Para construir e iniciar nuevamente los tres servicios:

```bash
docker compose up --build -d
docker compose ps
```

No agregar `-v` a `docker compose down` cuando se deban conservar los datos.

## 10. Incidencias corregidas y mejoras

- **R-01:** se corrigió el comando de inicio de Uvicorn, de `app:app` a `main:app`.
- **R-02:** se agregó `psycopg2-binary` para permitir la conexión de SQLAlchemy con PostgreSQL.
- **R-03:** se reemplazó `localhost` por el nombre del servicio `db` en la URL de conexión.
- **R-04:** se corrigieron las credenciales de PostgreSQL y se externalizaron mediante variables de entorno.
- **R-05:** se corrigió el destino del proxy Nginx para utilizar `backend:8000`.
- **R-06:** se corrigió la publicación del puerto del frontend a `8080:80`.
- **R-07:** se agregó el volumen persistente `postgres_data` y la configuración mediante `.env` / `.env.example`.
- **R-08:** se agregaron healthchecks para PostgreSQL, backend y frontend.
- **R-09:** se documentó el fallo controlado del backend y su posterior recuperación.

El detalle de síntomas, evidencias, causas raíz, cambios, justificaciones, verificaciones e impactos se encuentra en [DIAGNOSTICO.md](DIAGNOSTICO.md). Las capturas de Codespaces se encuentran en `evidencias/capturas/` y las salidas de comandos en `evidencias/`.

## 11. Resultado

Se recuperó la aplicación original y se verificaron la comunicación entre los tres servicios, la salud de la API, la persistencia de datos tras recrear PostgreSQL y la recuperación después de detener el backend. El historial de GitHub conserva los cambios de diagnóstico, reparación, mejora y documentación.
