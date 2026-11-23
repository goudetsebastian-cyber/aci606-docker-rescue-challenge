# ACI606 — Especialidad II: Cloud Computing

## Evaluación 2 · Docker Rescue Challenge

**Integrantes:** pendiente de completar  
**Sección:** pendiente de completar  
**Fecha:** 23 de noviembre de 2026  
**Estado de entrega:** reparación y validación estática preparadas; ejecución Docker y publicación GitHub pendientes de completar en un entorno autenticado con Docker.

---

## 1. Resumen ejecutivo

El starter conserva la arquitectura indicada por la cátedra (Nginx, FastAPI y PostgreSQL), pero llega con fallas de arranque, acceso a la base de datos, direccionamiento interno y publicación del puerto. La inspección también detectó que no se habían incorporado volumen nombrado, configuración externa ni comprobaciones de salud.

Se corrigieron las rutas de módulo y base de datos, se añadió el driver PostgreSQL, se ajustó el ruteo Nginx y se alineó el puerto del host con el puerto real del contenedor. Luego se incorporaron `.env.example`, un volumen PostgreSQL, healthchecks por servicio, dependencias condicionadas a salud, un README de reproducción y un diagrama de arquitectura.

La validación estática concluyó correctamente para YAML, sintaxis Python y JavaScript. No se ejecutó Docker Compose: el entorno de trabajo no reconoce el comando `docker` y no contiene Docker Desktop en las ubicaciones verificadas. Por ello, construcción, arranque, comunicación, persistencia y fallo controlado se registran como pendientes; no se reportan como pruebas aprobadas.

## 2. Arquitectura inicial

El navegador debía conectarse a Nginx por el puerto 8080; Nginx debía reenviar `/api` a FastAPI; y FastAPI debía consultar PostgreSQL. Compose proporciona una red interna con descubrimiento por nombre de servicio. En la entrega original, Nginx apuntaba a `localhost`, Compose publicaba `8080:8080` aunque el contenedor escucha en 80, y `DATABASE_URL` usaba `localhost` con una contraseña distinta de la configurada en PostgreSQL.

La ruta de inicio del backend indicaba `app:app`, pero el módulo entregado es `main.py` y el objeto ASGI se llama `app`. Además, `requirements.txt` no instalaba un driver DBAPI para PostgreSQL y el backend podía iniciar antes de que la base aceptara conexiones.

Diagrama inicial simplificado: `Navegador → Nginx:80 → FastAPI:8000 → PostgreSQL:5432`. No existía volumen nombrado para el directorio de datos.

## 3. Metodología de diagnóstico

Se aplicó la secuencia solicitada por la clase: síntoma, evidencia, causa raíz, corrección y verificación. Primero se inventariaron los ocho archivos entregados; después se contrastaron el `Dockerfile`, `requirements.txt`, `compose.yaml`, `nginx.conf` y el código Python. Los hallazgos que dependen de una ejecución se distinguen de los que se pueden demostrar leyendo los archivos.

Comandos de inspección previstos en un entorno Docker: `docker compose config`, `docker compose ps`, `docker compose logs <servicio>`, `docker inspect <contenedor>`, `docker network inspect <red>` y `docker volume inspect <volumen>`. En esta sesión `docker --version` no fue reconocido por PowerShell, por lo que esos comandos no se ejecutaron.

## 4. Bitácora de reparaciones

### R-01 · Módulo ASGI incorrecto

- **Síntoma esperado:** el proceso Uvicorn no encuentra el módulo `app` y el backend no inicia.
- **Evidencia:** `backend/Dockerfile` terminaba con `uvicorn app:app`; el único archivo de aplicación recibido era `backend/main.py`, donde se declara `app = FastAPI(...)`.
- **Causa raíz:** el nombre de módulo del comando no corresponde al archivo Python.
- **Elemento y reparación:** `backend/Dockerfile`; cambiar `app:app` a `main:app`.
- **Justificación:** Uvicorn importa el objeto `app` desde el módulo `main` que contiene el archivo suministrado.
- **Verificación:** pendiente ejecutar `docker compose up --build -d`, revisar `docker compose logs backend` y consultar `/api/health`.
- **Impacto:** sin la API, Nginx no puede entregar salud ni registros.

### R-02 · Driver PostgreSQL ausente

- **Síntoma esperado:** la importación del motor SQLAlchemy falla por falta de un driver PostgreSQL en la imagen.
- **Evidencia:** el archivo original de dependencias declaraba FastAPI, Uvicorn y SQLAlchemy, pero ningún DBAPI de PostgreSQL; el código construye un engine PostgreSQL al importar.
- **Causa raíz:** declarar SQLAlchemy no instala por sí solo el driver de base de datos requerido.
- **Elemento y reparación:** `backend/requirements.txt`; añadir `psycopg[binary]==3.2.3` y usar explícitamente `postgresql+psycopg://` en `DATABASE_URL`.
- **Justificación:** el driver y el dialecto explícitos corresponden a Psycopg 3 y evitan una selección implícita distinta.
- **Verificación:** pendiente construir la imagen y confirmar el inicio del backend sin errores de importación; después, consultar `/api/health`.
- **Impacto:** sin driver, el backend no puede abrir conexiones ni crear la tabla.

### R-03 · URL de base de datos inconsistente

- **Síntoma esperado:** conexión rechazada o autenticación fallida al iniciar la API.
- **Evidencia:** la URL original usaba `localhost` y `wrongpass`; el servicio `db` se llama `db` y declara `POSTGRES_PASSWORD=rescue123`.
- **Causa raíz:** dentro de un contenedor, `localhost` apunta al propio contenedor, y la contraseña no coincide con la inicialización de PostgreSQL.
- **Elemento y reparación:** `compose.yaml` y `.env.example`; usar `db:5432`, credenciales coincidentes y externalizar la URL.
- **Justificación:** Compose resuelve servicios mediante DNS interno por nombre y el cliente debe usar las credenciales configuradas en la base.
- **Verificación:** pendiente esperar salud de `db` y obtener HTTP 200 en `/api/health` con `database=reachable`.
- **Impacto:** el backend queda sin persistencia ni acceso a registros.

### R-04 · Proxy de Nginx hacia localhost

- **Síntoma esperado:** las rutas `/api/*` devuelven error de upstream aunque la API esté activa.
- **Evidencia:** `frontend/nginx.conf` apuntaba a `http://localhost:8000`; frontend y backend son contenedores distintos.
- **Causa raíz:** `localhost` dentro de Nginx representa al contenedor frontend, no al servicio FastAPI.
- **Elemento y reparación:** `frontend/nginx.conf`; cambiar el upstream a `http://backend:8000/` y conservar el prefijo `/api/` en `location` para que Nginx lo elimine al reenviar.
- **Justificación:** `backend` es el nombre estable de servicio en la red Compose; el puerto de comunicación es el interno 8000.
- **Verificación:** pendiente comprobar `/api/health`, `GET /api/items` y `POST /api/items` desde el host.
- **Impacto:** el frontend carga, pero no puede consultar ni guardar información.

### R-05 · Puerto de host y contenedor desalineados

- **Síntoma esperado:** el host intenta conectarse al puerto 8080 del contenedor, donde Nginx no escucha.
- **Evidencia:** Compose publicaba `8080:8080`; el Dockerfile del frontend expone 80 y `nginx.conf` escucha en 80.
- **Causa raíz:** el formato `puerto_host:puerto_contenedor` requiere el puerto real de escucha del contenedor.
- **Elemento y reparación:** `compose.yaml`; usar `${FRONTEND_PORT:-8080}:80`.
- **Justificación:** el host mantiene una URL sencilla en 8080 y el tráfico llega al puerto interno de Nginx.
- **Verificación:** pendiente abrir `http://localhost:8080/` y consultar `/api/health`.
- **Impacto:** el navegador no alcanza ni siquiera el frontend.

### R-06 · Orden de arranque sin comprobación de disponibilidad

- **Síntoma esperado:** fallos intermitentes de backend al intentar conectarse a PostgreSQL durante el evento de inicio.
- **Evidencia:** la aplicación crea la tabla al arrancar y el Compose original solo ordenaba el inicio con la forma corta de `depends_on`, sin healthcheck.
- **Causa raíz:** que un contenedor esté iniciado no significa que el servicio interno esté listo para aceptar consultas.
- **Elemento y reparación:** `compose.yaml`; healthcheck `pg_isready`, salud funcional de FastAPI y dependencias con `condition: service_healthy`.
- **Justificación:** Compose espera disponibilidad de la base antes del backend y del backend saludable antes del frontend.
- **Verificación:** pendiente comprobar `docker compose ps` y logs durante un arranque limpio.
- **Impacto:** el despliegue puede fallar por una carrera de inicio incluso con credenciales correctas.

## 5. Mejoras incorporadas y arquitectura final

El flujo final queda así: `Navegador → frontend Nginx (host 8080 → contenedor 80) → backend FastAPI (red interna, puerto 8000) → db PostgreSQL (red interna, puerto 5432)`. El frontend es el único servicio publicado al host. La configuración de acceso se lee desde `.env`; PostgreSQL guarda sus datos en `postgres_data`.

- **M-01 Persistencia:** volumen nombrado `postgres_data` en `/var/lib/postgresql/data`.
- **M-02 Configuración externa:** `.env.example` muestra `FRONTEND_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` y `DATABASE_URL`; `.env` se excluye de Git.
- **M-03 Salud y dependencias:** checks para PostgreSQL, FastAPI (incluye consulta a DB) y Nginx; `depends_on` usa `service_healthy`.

El diagrama actualizado se entrega como `docs/arquitectura.svg` y está incluido en el PDF.

## 6. Pruebas de validación

| Comprobación | Estado | Evidencia |
|---|---|---|
| Parseo YAML | Aprobada | PyYAML cargó `compose.yaml`; se comprobaron los tres servicios, mapeo 8080:80, dependencias saludables, volumen y healthchecks. |
| Sintaxis Python | Aprobada | `python -m py_compile backend/main.py`. |
| Sintaxis JavaScript | Aprobada | Script extraído de `frontend/index.html` aceptado por `node --check`. |
| Validación Compose nativa | Pendiente | Docker CLI/Engine no disponible (`docker` no reconocido). |
| Arranque limpio y comunicación | Pendiente | Requiere `docker compose up --build -d` en Docker/Codespaces. |
| Persistencia tras recreación | Pendiente | Requiere insertar registro, ejecutar `docker compose down` y recrear servicios. |
| Fallo controlado y recuperación | Pendiente | Requiere detener y reactivar backend con Compose. |

La aprobación de parseo YAML confirma la sintaxis del documento, no sustituye `docker compose config` ni una ejecución real.

## 7. Procedimiento reproducible

Desde la raíz del proyecto:

```bash
cp .env.example .env
docker compose config --quiet
docker compose up --build -d
docker compose ps
docker compose logs --tail=100
curl -i http://localhost:8080/api/health
curl -X POST http://localhost:8080/api/items -H "Content-Type: application/json" -d '{"name":"registro-demo"}'
docker compose down
docker compose up --build -d
curl http://localhost:8080/api/items
docker compose stop backend
curl -i http://localhost:8080/api/health
docker compose logs --tail=50 backend frontend
docker compose start backend
docker compose ps
curl -i http://localhost:8080/api/health
```

El registro debe permanecer tras `down`/`up` porque el volumen nombrado no se elimina. `docker compose down -v` sí elimina el volumen y los datos. Para Windows PowerShell, README.md incluye el comando `Copy-Item` equivalente.

## 8. Conclusiones, trazabilidad y pendientes de entrega

Se corrigieron seis incidencias a partir de inconsistencias verificables en los archivos suministrados y se incorporaron las cuatro mejoras obligatorias de la pauta (persistencia, configuración externa, healthcheck y README reproducible). La configuración estática es coherente y el árbol de trabajo queda registrado en commits locales.

La limitación principal es de entorno: en esta sesión no hay Docker CLI ni Engine. Por lo mismo, no se presenta una falsa evidencia de build, tráfico, persistencia o recuperación. Antes de entregar, falta ejecutar la matriz dinámica en Docker o Codespaces y adjuntar su salida real.

**Repositorio GitHub:** pendiente de publicar. La cuenta conectada no ofreció una acción de creación de repositorio y la sesión web solicitó iniciar sesión. El proyecto está preparado en Git local con la rama `main`; no se incluye un enlace inventado.

**Campos que debe completar el equipo antes de entregar:** integrantes y sección. Una vez autenticada la sesión y creado el repositorio, añadir su URL al README y a esta sección, publicar la rama `main` y conceder acceso al docente si el repositorio es privado.

### Fuentes

- `ACI606_Clase_Docker_Final.pdf`, láminas 15, 22–25 y 29–30.
- `Catedra 2.pdf`, secciones 6–10, páginas 3–5.
- [Docker Docs — Networking in Compose](https://docs.docker.com/compose/how-tos/networking/)
- [Docker Docs — Control startup and shutdown order](https://docs.docker.com/compose/how-tos/startup-order/)
- [Docker Docs — Running containers and port publishing](https://docs.docker.com/engine/containers/run/)
- [Docker Docs — Volumes](https://docs.docker.com/engine/storage/volumes/)
- [SQLAlchemy 2.0 — PostgreSQL dialect](https://docs.sqlalchemy.org/en/20/dialects/postgresql.html)
- [Psycopg — Installation](https://www.psycopg.org/psycopg3/docs/basic/install.html)