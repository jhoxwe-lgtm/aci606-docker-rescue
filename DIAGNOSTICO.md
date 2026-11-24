# Diagnóstico y bitácora de reparaciones - ACI606 Docker Rescue Challenge

Este documento resume el proceso sobre el starter original. Para cada incidencia se registra síntoma, evidencia, causa raíz, elemento modificado, reparación, justificación, verificación e impacto, según la pauta de Evaluación 2.

## R-01 - Comando de inicio del backend
- **Síntoma:** backend termina al iniciar; Uvicorn informa `Could not import module "app"`.
- **Evidencia:** `docker compose logs --no-color --tail=100 backend`; respaldo: `evidencias/R-01-backend-logs.txt`.
- **Causa raíz:** el Dockerfile ejecutaba `uvicorn app:app`, pero el código se encuentra en `backend/main.py` y declara `app = FastAPI(...)`.
- **Archivo:** `backend/Dockerfile`.
- **Reparación:** cambiar el módulo de `app:app` a `main:app`.
- **Justificación:** Uvicorn importa `main.py` como módulo `main` y resuelve el objeto ASGI `app`.
- **Verificación:** logs con `Application startup complete` y `Uvicorn running on http://0.0.0.0:8000`; backend posteriormente `healthy`.
- **Impacto:** sin backend no existe API para frontend ni conexión a PostgreSQL.

## R-02 - Driver PostgreSQL ausente
- **Síntoma:** error `ModuleNotFoundError: No module named 'psycopg2'` al crear el motor SQLAlchemy.
- **Evidencia:** `docker compose logs --no-color --tail=50 backend`; respaldo `evidencias/R-02-backend-driver.txt`.
- **Causa raíz:** faltaba el controlador DBAPI de PostgreSQL en los requisitos.
- **Archivo:** `backend/requirements.txt`.
- **Reparación:** agregar `psycopg2-binary==2.9.10`.
- **Justificación:** SQLAlchemy utiliza el controlador `psycopg2` para hablar con PostgreSQL.
- **Verificación:** reconstrucción de imagen; backend inicia y `/api/health` informa base `reachable`.
- **Impacto:** backend no puede abrir conexiones ni operar con la base de datos.

## R-03 - Host de base de datos incorrecto
- **Síntoma:** conexión rechazada al intentar conectar a `localhost:5432` desde backend.
- **Evidencia:** logs de backend y revisión del Compose original; respaldo `evidencias/R-03-db-localhost.txt`.
- **Causa raíz:** dentro de un contenedor, `localhost` identifica al propio contenedor, no al servicio `db`.
- **Archivo:** `compose.yaml`, variable `DATABASE_URL`.
- **Reparación:** cambiar host `localhost` por `db`.
- **Justificación:** Compose resuelve nombres de servicio en la red interna.
- **Verificación:** `curl -i http://127.0.0.1:8080/api/health` devuelve HTTP 200 y `database: reachable` una vez corregida la contraseña R-04.
- **Impacto:** API no podría consultar ni insertar datos.

## R-04 - Contraseña de PostgreSQL incorrecta
- **Síntoma:** `FATAL: password authentication failed for user "appuser"`.
- **Evidencia:** logs de backend; respaldo `evidencias/R-04-db-authentication.txt`.
- **Causa raíz:** `DATABASE_URL` usaba `wrongpass`, pero PostgreSQL se inicializó con `rescue123`.
- **Archivo:** `compose.yaml`; luego se externalizó con `.env`.
- **Reparación:** alinear la contraseña de la URL con `POSTGRES_PASSWORD` y parametrizarla.
- **Justificación:** usuario, contraseña y base deben coincidir entre backend y PostgreSQL.
- **Verificación:** backend inicia; `/api/health` responde HTTP 200 y `database: reachable`.
- **Impacto:** PostgreSQL rechaza cada conexión del backend.

## R-05 - Upstream de Nginx incorrecto
- **Síntoma:** el frontend no logra consumir las rutas `/api/`.
- **Evidencia:** `cat frontend/nginx.conf`; respaldo de configuración y corrección `evidencias/R-05-R-06-frontend-config.txt` y `evidencias/R-05-nginx-upstream-fixed.txt`.
- **Causa raíz:** `proxy_pass` apuntaba a `localhost:8000`, que dentro del contenedor Nginx es el propio frontend.
- **Archivo:** `frontend/nginx.conf`.
- **Reparación:** usar `proxy_pass http://backend:8000/`.
- **Justificación:** `backend` es el nombre de servicio que Compose resuelve en la red interna.
- **Verificación:** `docker compose exec frontend nginx -t` y `docker compose exec frontend wget -qO- http://127.0.0.1/api/health` devuelven configuración válida y `database: reachable`.
- **Impacto:** frontend no podría llamar a la API aunque FastAPI estuviera disponible.

## R-06 - Mapeo de puerto frontend incorrecto
- **Síntoma:** el servicio no era accesible por el puerto publicado esperado.
- **Evidencia:** Nginx escucha en 80; `frontend/Dockerfile` expone 80; Compose original publicaba `8080:8080`; respaldo `evidencias/R-06-port-diagnosis.txt`.
- **Causa raíz:** el puerto interno del mapeo no correspondía al puerto de escucha de Nginx.
- **Archivo:** `compose.yaml`.
- **Reparación:** cambiar a `8080:80`.
- **Justificación:** Docker publica el puerto host 8080 hacia el puerto 80 del contenedor.
- **Verificación:** `docker compose ps` muestra `0.0.0.0:8080->80/tcp`; `curl -i http://127.0.0.1:8080/api/health` devuelve HTTP 200 tras completar las reparaciones.
- **Impacto:** el navegador no podría acceder al frontend mediante el puerto publicado.

## R-07 - Volumen PostgreSQL y configuración externalizada
- **Síntoma:** Compose original no declaraba volumen de datos y contenía valores de conexión rígidos.
- **Evidencia:** inspección del Compose original; respaldo SQL, registro de prueba antes/después y volumen inspeccionado en `evidencias/R-07-*.txt`.
- **Causa raíz:** sin volumen, el almacenamiento queda ligado al contenedor; sin externalización, los valores son difíciles de cambiar y se corre el riesgo de versionar secretos.
- **Archivos:** `compose.yaml`, `.env.example`, `.gitignore`.
- **Reparación:** crear volumen `postgres_data` montado en `/var/lib/postgresql/data`; parametrizar `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` y `APP_PORT`; excluir `.env` del repositorio.
- **Justificación:** el volumen sobrevive a la recreación del contenedor; `.env` separa configuración local del código; `.env.example` permite reproducir las variables sin revelar la clave local.
- **Verificación:** se creó el registro `Prueba de persistencia Docker` (ID 1), se recreó `db` con `docker compose up -d --force-recreate db`, `pg_isready` aceptó conexiones y el mismo registro apareció en `/api/items` después de la recreación.
- **Impacto:** sin volumen se podría perder información; sin variables externas se mantendrían credenciales rígidas en Compose.

## R-08 - Healthchecks
- **Síntoma:** `depends_on` simple no comprobaba disponibilidad real de los servicios.
- **Evidencia:** Compose original sin healthchecks; configuración final verificada con `docker compose config -q`; estado guardado en `evidencias/R-08-estado-servicios.txt`.
- **Causa raíz:** el orden de inicio no demuestra que un servicio responda a solicitudes funcionales.
- **Archivo:** `compose.yaml`.
- **Reparación:** healthcheck PostgreSQL (`pg_isready`), backend (petición Python a `/health`) y frontend (`wget --spider /`); `depends_on` con `condition: service_healthy`.
- **Justificación:** los servicios dependientes esperan a que el proveedor esté funcional y Docker marca el estado real de salud.
- **Verificación:** `docker compose ps` mostró `frontend`, `backend` y `db` como `healthy`.
- **Impacto:** sin healthchecks, servicios parcialmente inicializados podrían mostrarse como ejecutándose y dificultar diagnóstico/arranque reproducible.

## R-09 - Fallo controlado y recuperación
- **Síntoma:** al detener backend, Nginx responde HTTP 502 para `/api/health`.
- **Evidencia:** `docker compose stop backend`, `docker compose ps -a`, `curl -i --max-time 5 http://127.0.0.1:8080/api/health`; respaldo `evidencias/R-09-fallo-controlado.txt`.
- **Causa raíz:** Nginx no puede reenviar solicitudes API cuando el proceso FastAPI está detenido.
- **Elemento intervenido:** estado del servicio `backend`, sin cambiar archivos de código.
- **Reparación/recuperación:** ejecutar `docker compose start backend` y esperar el healthcheck.
- **Justificación:** se utiliza Compose para recuperar el mismo servicio manteniendo la red y el volumen de datos.
- **Verificación:** `docker compose ps` mostró los tres servicios `healthy`; `/api/health` volvió a HTTP 200 y `/api/items` conservó el registro de prueba. Evidencia: `evidencias/R-09-recuperacion.txt`.
- **Impacto:** si el backend permanece detenido, la interfaz no puede consultar ni registrar elementos.

## Evidencias relacionadas

- R-01 a R-06: archivos `evidencias/R-01-*` a `evidencias/R-06-*`.
- R-07: `R-07-registro-creado.txt`, `R-07-registros-antes.txt`, `R-07-registros-despues.txt`, `R-07-inspeccion-volumen.txt`.
- R-08: `R-08-estado-servicios.txt`.
- R-09: `R-09-fallo-controlado.txt`, `R-09-recuperacion.txt`.
- Capturas de Codespaces: `evidencias/capturas/`.
