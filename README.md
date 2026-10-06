# Taller de Docker

Este taller tiene como objetivo aprender a **crear imágenes propias, ejecutar contenedores y componer aplicaciones de varios servicios** con Docker y Docker Compose.
Se parte de una aplicación web sencilla en Python con una base de datos Redis, se escala detrás de un balanceador de carga y, al final, se aplica lo aprendido a una aplicación **Java / Spring Boot** como la del proyecto del curso.

---

## Objetivo General

Comprender y aplicar los conceptos de **imagen, contenedor, capa, volumen y red** de Docker, escribiendo **Dockerfiles** con buenas prácticas (caché de capas, `.dockerignore`, usuario sin privilegios, *healthcheck*, *multi-stage*), orquestando servicios con **Docker Compose**, distribuyendo la carga con **Traefik** y publicando imágenes en un **registro**.

---

## Índice

- [Conceptos clave](#conceptos-clave)
- [CONOCE EL TALLER](#conoce-el-taller)
  - [Requisitos](#requisitos)
  - [Estructura de trabajo](#estructura-de-trabajo)
- [PARTE 1: IMÁGENES PROPIAS](#parte-1-imágenes-propias)
  - [Paso 1: Mi primer Dockerfile](#paso-1-mi-primer-dockerfile)
  - [Paso 2: Una aplicación web en un contenedor](#paso-2-una-aplicación-web-en-un-contenedor)
  - [Paso 3: Construir y ejecutar la imagen](#paso-3-construir-y-ejecutar-la-imagen)
  - [Paso 4: Caché de capas](#paso-4-caché-de-capas)
- [PARTE 2: DOCKER COMPOSE](#parte-2-docker-compose)
  - [Paso 5: Aplicación web con Redis](#paso-5-aplicación-web-con-redis)
  - [Paso 6: Balanceo de carga con Traefik](#paso-6-balanceo-de-carga-con-traefik)
- [PARTE 3: COMPARTIR IMÁGENES](#parte-3-compartir-imágenes)
- [PARTE 4: CONTENERIZAR UNA APLICACIÓN JAVA](#parte-4-contenerizar-una-aplicación-java)
- [Escaneo de vulnerabilidades (opcional)](#escaneo-de-vulnerabilidades-opcional)
- [Problemas frecuentes](#problemas-frecuentes)
- [Buenas prácticas](#buenas-prácticas)
- [Para entregar](#para-entregar-con-este-taller)
- [Cómo usar esta guía para tu proyecto](#cómo-usar-esta-guía-para-tu-proyecto)
- [Resumen del Taller](#hagamos-un-resumen)
- [Conclusión](#conclusión)
- [Recursos recomendados](#recursos-recomendados)
- [Créditos y uso académico](#créditos-y-uso-académico)
- [Licencia](#licencia-de-uso)

---

## Conceptos clave

- **Imagen**
Plantilla de solo lectura con todo lo necesario para ejecutar una aplicación: sistema base, librerías, código y configuración.
Ejemplo: `python:3.14-slim` o `friendlyhello`, la imagen que se construye en este taller.

- **Contenedor**
Instancia en ejecución de una imagen. Es un proceso aislado que comparte el kernel del sistema anfitrión. Se puede crear, detener y eliminar sin afectar la imagen.

- **Capa**
Cada instrucción del Dockerfile que modifica archivos (`COPY`, `RUN`) genera una capa. Docker reutiliza (cachea) las capas que no cambiaron, lo que hace más rápidos los siguientes *builds*.

- **Dockerfile y *build context***
El Dockerfile contiene las instrucciones para construir la imagen. El *build context* es el directorio que se envía a Docker al construir: solo lo que está en ese directorio se puede copiar a la imagen.

- **Volumen**
Almacenamiento que vive fuera del contenedor. Los datos de un volumen se conservan aunque el contenedor se elimine.

- **Docker Compose**
Herramienta para definir y ejecutar aplicaciones de varios contenedores con un archivo YAML. Los servicios de un mismo archivo comparten una red y se encuentran por su nombre.

- **Registro**
Servidor que almacena y distribuye imágenes. Docker Hub es el registro público por defecto; GitHub Container Registry (GHCR) es otra opción.

---

## CONOCE EL TALLER

### Requisitos

- **Docker Desktop** (Windows o macOS) o **Docker Engine** (Linux). En Windows, Docker Desktop usa WSL 2.
- **Docker Compose v2**, que viene incluido en Docker Desktop. El comando es `docker compose` (con espacio); `docker-compose` con guion es la versión 1, que ya no se mantiene.

Verifica la instalación:

```bash
docker version
docker compose version
```

`docker version` debe mostrar una sección **Client** y otra **Server**. Si solo aparece **Client** junto con un error de conexión (en Windows, por ejemplo, `open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified`), Docker Desktop no está iniciado: ábrelo y espera a que termine de arrancar.

### Estructura de trabajo

Durante el taller crearás dos carpetas de trabajo:

```text
hello-world/                     # Paso 1
 ├─ Dockerfile
 └─ hello
friendlyhello/                   # Pasos 2 a 6
 ├─ app.py                       # aplicación web (Flask)
 ├─ requirements.txt             # dependencias con versión fija
 ├─ .dockerignore                # archivos que NO se envían al build
 ├─ Dockerfile
 ├─ docker-compose.yaml          # web + redis
 ├─ docker-compose.traefik.yaml  # web (varias réplicas) + redis + traefik
 └─ data/                        # la crea Compose: datos de Redis
```

La [Parte 4](#parte-4-contenerizar-una-aplicación-java) se hace directamente en el repositorio del proyecto del equipo.

---

## PARTE 1: IMÁGENES PROPIAS

Docker permite crear imágenes propias. Aunque podrían construirse desde cero, lo habitual es partir de una **imagen base**: una distribución (Debian, Ubuntu, Alpine) o una imagen con un lenguaje ya instalado (Python, Java).

### Paso 1: Mi primer Dockerfile

Crea el *build context*:

```bash
mkdir hello-world
cd hello-world
echo "hello" > hello
```

Dentro de esa carpeta crea un archivo llamado `Dockerfile` con este contenido:

```dockerfile
FROM busybox:1.37
COPY hello /
RUN cat /hello
```

| Directiva | Explicación |
|-----------|-------------|
| `FROM` | Imagen base sobre la que se construye la nueva imagen. Se indica una versión fija (`1.37`) para que el resultado no cambie con el tiempo. |
| `COPY` | Copia un archivo del *build context* a la imagen. |
| `RUN` | Ejecuta un comando **durante la construcción** de la imagen. |

Construye la imagen. El parámetro `-t` le asigna un nombre y una versión, y el `.` indica que el *build context* es el directorio actual:

```bash
docker build -t helloapp:v1 .
```

Docker usa **BuildKit** para construir. La salida muestra cada paso numerado y, por defecto, oculta lo que imprime `RUN`. Para ver el `hello` impreso por `cat`, agrega `--progress=plain`:

```bash
docker build --progress=plain -t helloapp:v1 .
```

```text
#8 [3/3] RUN cat /hello
#8 0.385 hello
...
#9 naming to docker.io/library/helloapp:v1 done
```

Comprueba que la imagen quedó instalada:

```bash
docker images helloapp
```

```text
REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
helloapp     v1    …              10 seconds ago   6.79MB
```

> El tamaño puede variar según la versión de Docker y el almacén de imágenes que use (Docker Desktop reciente usa *containerd* y reporta tamaños mayores que versiones anteriores).

### Paso 2: Una aplicación web en un contenedor

Ahora se contenerizará una aplicación web en Python que cuenta las visitas usando Redis. Crea una nueva carpeta:

```bash
mkdir friendlyhello
cd friendlyhello
```

**`app.py`**

```python
from flask import Flask
from redis import Redis, RedisError
from redis.backoff import NoBackoff
from redis.retry import Retry
import os
import socket

# Conexión a Redis: el nombre "redis" lo resuelve la red de Docker Compose.
# Timeouts cortos y sin reintentos para que la página responda aunque Redis no exista.
redis = Redis(host="redis", db=0,
              socket_connect_timeout=2, socket_timeout=2,
              retry=Retry(NoBackoff(), 0))

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)


@app.route("/health")
def health():
    # Lo usa el HEALTHCHECK del Dockerfile: no depende de Redis
    return "OK"
```

> **Sobre `retry=Retry(NoBackoff(), 0)`.** Las versiones recientes de la librería `redis` reintentan cada operación fallida varias veces (10 reintentos en la versión 8.x). Sin esta línea, cuando Redis no está disponible la página puede tardar más de un minuto y medio en responder.

**`requirements.txt`**: las dependencias con **versión fija**, para que la imagen se construya igual hoy y dentro de un año:

```text
Flask==3.1.3
redis==8.1.0
gunicorn==26.2.0
```

**`.dockerignore`**: lo que **no** se debe enviar al *build context*. Sin este archivo, `COPY . .` copiaría a la imagen los datos de Redis (`data/`) y los archivos de Compose:

```text
data/
docker-compose*.yaml
Dockerfile
.dockerignore
.git
__pycache__/
*.pyc
```

**`Dockerfile`**

```dockerfile
# Imagen base oficial de Python, con versión fija
FROM python:3.14-slim

# Directorio de trabajo dentro de la imagen
WORKDIR /app

# Primero solo las dependencias: esta capa se reutiliza mientras requirements.txt no cambie
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Después el código de la aplicación
COPY . .

# Usuario sin privilegios para ejecutar la aplicación
RUN useradd --system --no-create-home app
USER app

# Variable de entorno y puerto de la aplicación
ENV NAME=World
EXPOSE 8000

# Docker marca el contenedor como "unhealthy" si la aplicación deja de responder
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --start-interval=2s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health', timeout=2)"

# Servidor de producción (gunicorn) en lugar del servidor de desarrollo de Flask
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]
```

| Directiva | Explicación |
|-----------|-------------|
| `WORKDIR` | Directorio de trabajo para las instrucciones siguientes y para el contenedor. |
| `COPY requirements.txt .` + `RUN pip install` | Se instalan las dependencias **antes** de copiar el código. Así, al cambiar `app.py`, Docker reutiliza la capa de dependencias (ver el [Paso 4](#paso-4-caché-de-capas)). |
| `USER app` | El proceso del contenedor no corre como `root`. Si alguien compromete la aplicación, tiene menos permisos dentro del contenedor. |
| `ENV NAME=World` | Variable de entorno. Se usa la forma `clave=valor`; la forma antigua `ENV NAME World` genera la advertencia `LegacyKeyValueFormat`. |
| `EXPOSE 8000` | Documenta el puerto en el que escucha la aplicación. No publica el puerto por sí solo. Se usa el 8000 porque un usuario sin privilegios no puede abrir puertos menores a 1024. |
| `HEALTHCHECK` | Comando que Docker ejecuta periódicamente para saber si la aplicación responde. `--start-interval` acelera las comprobaciones mientras el contenedor arranca. |
| `CMD` | Comando que se ejecuta al iniciar el contenedor. Se usa **gunicorn** porque el servidor de Flask es solo para desarrollo. |

Para conocer todas las directivas visita la [documentación oficial de Dockerfile](https://docs.docker.com/reference/dockerfile/).

En total debes tener 4 archivos (`.dockerignore` es un archivo oculto):

```bash
ls -A
```

```text
.dockerignore  app.py  Dockerfile  requirements.txt
```

### Paso 3: Construir y ejecutar la imagen

Antes de construir, Docker puede revisar el Dockerfile y avisar de malas prácticas:

```bash
docker build --check .
```

```text
Check complete, no warnings found.
```

Construye la imagen y comprueba que existe:

```bash
docker build -t friendlyhello .
docker images friendlyhello
```

Ejecuta un contenedor publicando el puerto 8000 del contenedor en el puerto 4000 de tu equipo:

```bash
docker run --rm -p 4000:8000 friendlyhello
```

| Parámetro | Explicación |
|-----------|-------------|
| `-p 4000:8000` | Publica el puerto: `puerto_del_equipo:puerto_del_contenedor`. |
| `--rm` | Elimina el contenedor cuando se detiene. Los contenedores de prueba son desechables; los datos que deben persistir van en volúmenes. |

Abre http://localhost:4000. Verás algo como:

```text
Hello World!
Hostname: 06881a4f6550
Visits: cannot connect to Redis, counter disabled
```

El *hostname* es el identificador del contenedor. El contador no funciona porque todavía no hay un Redis. La primera respuesta puede tardar unos segundos: el contenedor intenta resolver el nombre `redis` y esa búsqueda falla.

En otra terminal, revisa el estado del contenedor y el usuario con el que corre la aplicación:

```bash
docker ps
docker exec <CONTAINER_ID> id
```

```text
STATUS
Up 40 seconds (healthy)

uid=999(app) gid=999(app) groups=999(app)
```

Presiona `Ctrl+C` en la primera terminal para detener el contenedor.

### Paso 4: Caché de capas

1. Cambia el texto `Hello` por `Hola` en `app.py`.
2. Vuelve a construir con `docker build --progress=plain -t friendlyhello .`
3. Observa que el paso `RUN pip install` aparece como `CACHED`: las dependencias no se reinstalan porque `requirements.txt` no cambió.

> **Para discutir:** ¿qué pasaría si el Dockerfile tuviera `COPY . .` **antes** de `RUN pip install`? Pruébalo y compara el tiempo de construcción.

---

## PARTE 2: DOCKER COMPOSE

### Paso 5: Aplicación web con Redis

La aplicación necesita un servicio de Redis. En lugar de levantar cada contenedor a mano, se describen los dos servicios en un archivo de Compose.

**`docker-compose.yaml`**

```yaml
services:
  web:
    build: .
    ports:
      - "4000:8000"
    depends_on:
      redis:
        condition: service_healthy
  redis:
    image: redis:8.2
    command: redis-server --appendonly yes
    volumes:
      - ./data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

| Elemento | Explicación |
|----------|-------------|
| `build: .` | El servicio `web` se construye con el Dockerfile de la carpeta. El servicio `redis` usa una imagen ya publicada (`image:`). |
| `volumes: ./data:/data` | Los datos de Redis se guardan en la carpeta `data/` de tu equipo y sobreviven a la eliminación del contenedor. |
| `healthcheck` + `depends_on: condition: service_healthy` | `web` no arranca hasta que Redis responde. Un `depends_on` sin condición solo espera a que el contenedor **inicie**, no a que esté listo. |

Levanta la aplicación:

```bash
docker compose up -d --build
```

```text
 Container friendlyhello-redis-1  Started
 Container friendlyhello-redis-1  Waiting
 Container friendlyhello-redis-1  Healthy
 Container friendlyhello-web-1  Started
```

Abre http://localhost:4000 y recarga varias veces: ahora el contador de visitas aumenta. Revisa los servicios:

```bash
docker compose ps
```

> Compose v2 nombra los contenedores con guiones: `friendlyhello-web-1`, `friendlyhello-redis-1`. En tutoriales antiguos (Compose v1) aparecen con guion bajo: `friendlyhello_web_1`.

Detén la aplicación. Los datos de `data/` se conservan para la siguiente ejecución:

```bash
docker compose down
```

### Paso 6: Balanceo de carga con Traefik

Ahora se ejecutarán **varias réplicas** del servicio `web` detrás de un balanceador de carga. Crea otro archivo de Compose:

**`docker-compose.traefik.yaml`**

```yaml
services:
  web:
    build: .
    depends_on:
      redis:
        condition: service_healthy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`localhost`)"
      - "traefik.http.routers.web.entrypoints=web"
      - "traefik.http.services.web.loadbalancer.server.port=8000"
  redis:
    image: redis:8.2
    command: redis-server --appendonly yes
    volumes:
      - ./data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
  traefik:
    image: traefik:v3.7
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.web.address=:4000"
    ports:
      - "4000:4000"   # aplicación, a través del balanceador
      - "8080:8080"   # dashboard de Traefik (solo para pruebas locales)
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
```

Diferencias con el archivo anterior:

- El servicio `web` **ya no publica puertos**: solo Traefik recibe tráfico desde fuera.
- Traefik lee las **etiquetas** (`labels`) de los contenedores para saber a cuáles enviar tráfico, en qué puerto escuchan y con qué regla (`Host(\`localhost\`)`).
- Traefik solo envía tráfico a réplicas en estado `healthy`; por eso es importante el `HEALTHCHECK` del Dockerfile.

Levanta la aplicación con **5 réplicas** de `web`:

```bash
docker compose -f docker-compose.traefik.yaml up -d --build --scale web=5
docker compose -f docker-compose.traefik.yaml ps
```

Espera unos segundos a que las réplicas aparezcan como `healthy` y abre **http://localhost:4000**. Al recargar, el *hostname* cambia: cada petición la atiende una réplica distinta, y el contador sigue aumentando porque todas comparten el mismo Redis. Compara los *hostnames* con la columna `CONTAINER ID` de `docker ps`.

El dashboard de Traefik está en http://localhost:8080.

> **Usa `localhost`, no `127.0.0.1`.** La regla del router es `Host(\`localhost\`)`, así que una petición a `http://127.0.0.1:4000` responde `404 page not found`.

> **Alcance de la demostración.** Las 5 réplicas corren en la misma máquina: si esa máquina falla, fallan todas. En producción las réplicas se distribuyen en varios nodos con un orquestador como Kubernetes. Además, `--api.insecure=true` deja el dashboard sin autenticación y montar `docker.sock` le da a Traefik control sobre Docker: ambas cosas solo son aceptables en un entorno local de pruebas.

Detén todo:

```bash
docker compose -f docker-compose.traefik.yaml down
```

---

## PARTE 3: COMPARTIR IMÁGENES

Para compartir una imagen se usa un **registro**. En este taller se usa Docker Hub.

1. Crea una cuenta en [Docker Hub](https://hub.docker.com/).
2. Crea un repositorio con **Create repository**. El nombre del repositorio será el de la imagen: `friendlyhello`. Tu usuario es el *namespace* obligatorio de la imagen.
3. Inicia sesión desde la terminal:

   ```bash
   docker login
   ```

   > Docker Desktop guarda la credencial en el almacén de credenciales del sistema. En Linux sin un *credential helper* configurado, la contraseña queda **sin cifrar** en `~/.docker/config.json`: configura uno o ejecuta `docker logout` al terminar. Usa un *Personal Access Token* de Docker Hub en lugar de tu contraseña.

4. Construye la imagen con el nombre `usuario/repositorio` y, además, una etiqueta de versión:

   ```bash
   docker build -t <usuario>/friendlyhello:0.1.0 -t <usuario>/friendlyhello:latest .
   ```

   Si no se indica etiqueta, Docker usa `latest`. En la siguiente versión se sube el número: `0.2.0` y `latest` apuntarán a la misma imagen, y `0.1.0` seguirá disponible.

5. Publica la imagen:

   ```bash
   docker push <usuario>/friendlyhello:0.1.0
   docker push <usuario>/friendlyhello:latest
   ```

### Ejercicios

1. Cambia `docker-compose.yaml` para usar **tu imagen publicada** (`image: <usuario>/friendlyhello:0.1.0`) en lugar de `build: .`.
2. Cambia `docker-compose.yaml` para usar la imagen de un compañero.

---

## PARTE 4: CONTENERIZAR UNA APLICACIÓN JAVA

Esta parte se hace en el **repositorio del proyecto del equipo** (aplicación Spring Boot con Maven). Se usa un Dockerfile **multi-stage**: una etapa compila con Maven y la imagen final solo contiene el JRE y el `.jar`.

**`Dockerfile`** (en la raíz del proyecto, junto al `pom.xml`)

```dockerfile
# syntax=docker/dockerfile:1

# Etapa 1: compilar el .jar con Maven (las pruebas ya corrieron en el pipeline de CI)
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn -B -ntp -q dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -B -ntp -q package -DskipTests \
 && cp target/*.jar /app.jar

# Etapa 2: imagen final solo con el JRE y el .jar
FROM eclipse-temurin:21-jre
RUN groupadd --system app && useradd --system --gid app --no-create-home app
WORKDIR /app
COPY --from=build --chown=app:app /app.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

**`.dockerignore`**

```text
target/
.git/
.github/
.idea/
.vscode/
*.md
```

| Elemento | Explicación |
|----------|-------------|
| `AS build` / `COPY --from=build` | La primera etapa tiene JDK y Maven; la imagen final solo copia el `.jar`. Maven, el código fuente y el repositorio local de dependencias no llegan a producción. |
| `COPY pom.xml` antes de `COPY src` | Las dependencias se descargan en una capa propia que se reutiliza mientras el `pom.xml` no cambie. |
| `--mount=type=cache,target=/root/.m2` | Caché de BuildKit para las dependencias de Maven entre *builds*. |
| `-DskipTests` | Las pruebas ya se ejecutan en el pipeline de CI con `mvn verify`; no se repiten al construir la imagen. |
| `eclipse-temurin:21-jre` | Solo el entorno de ejecución. Usa la misma versión de Java del proyecto o una superior. |

Construye y ejecuta:

```bash
docker build -t proyecto:dev .
docker run --rm -p 8080:8080 proyecto:dev
```

> Si `target/` tiene más de un `.jar` (por ejemplo, uno de fuentes), ajusta `cp target/*.jar` al nombre exacto del archivo que genera `spring-boot-maven-plugin`.

---

## Escaneo de vulnerabilidades (opcional)

Una imagen puede tener vulnerabilidades conocidas en el sistema base o en las dependencias de la aplicación. [Trivy](https://trivy.dev/) las detecta:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL proyecto:dev
```

Como referencia, una aplicación de ejemplo con **Spring Boot 2.7.18** (versión sin soporte de código abierto) arrojó **0 hallazgos** en los paquetes del sistema base y **8 críticos y 40 altos** en las dependencias Java, la mayoría en el Tomcat embebido. La imagen base estaba limpia: el riesgo venía de las dependencias desactualizadas del framework.

Este escaneo se puede automatizar como una etapa del pipeline de CI (ver el tema de DevSecOps).

---

## Problemas frecuentes

| Síntoma | Causa y solución |
|---------|------------------|
| `docker version` no muestra **Server** | Docker Desktop no está iniciado. Ábrelo y espera a que arranque. |
| `404 page not found` en el Paso 6 | Se abrió `http://127.0.0.1:4000`. La regla de Traefik es `Host(\`localhost\`)`: usa `http://localhost:4000`. También puede aparecer si las réplicas aún no están `healthy`. |
| La página tarda varios segundos sin Redis | El contenedor intenta resolver el nombre `redis` y la búsqueda DNS falla. Es esperado en el Paso 3. |
| `Bind for 0.0.0.0:4000 failed: port is already allocated` | Otro contenedor o programa usa el puerto. Detén el contenedor anterior (`docker ps`, `docker stop`) o cambia el puerto del equipo (`-p 4001:8000`). |
| `LegacyKeyValueFormat` al construir | Se usó `ENV NAME World`. Cambia a `ENV NAME=World`. |
| Los datos de Redis aparecen dentro de la imagen | Falta el `.dockerignore` con `data/`. |
| Los contenedores se llaman `friendlyhello_web_1` en un tutorial | Es la nomenclatura de Compose v1. En Compose v2 es `friendlyhello-web-1`. |
| En Git Bash (Windows) falla un comando con rutas como `/var/run/docker.sock` o `/app` | Git Bash convierte esas rutas a rutas de Windows. Antepón `MSYS_NO_PATHCONV=1` al comando o ejecútalo en PowerShell. |

---

## Buenas prácticas

1. **Versiones fijas:** imagen base (`python:3.14-slim`, `redis:8.2`, `traefik:v3.7`) y dependencias (`requirements.txt`, `pom.xml`) con versión. `latest` cambia sin aviso.
2. **Orden de capas:** primero lo que cambia poco (dependencias), después lo que cambia mucho (código).
3. **`.dockerignore` siempre:** el *build context* debe contener solo lo necesario.
4. **Usuario sin privilegios:** `USER` distinto de `root` en la imagen final.
5. ***Multi-stage* para lenguajes compilados:** las herramientas de compilación no llegan a la imagen final.
6. **`HEALTHCHECK` sobre un endpoint propio**, que no dependa de servicios externos.
7. **Servidor de producción** (gunicorn, el servidor embebido de Spring Boot), nunca el servidor de desarrollo.
8. **Sin secretos en la imagen:** contraseñas y *tokens* se pasan como variables de entorno o *secrets* en tiempo de ejecución, nunca con `COPY` o `ENV` en el Dockerfile.
9. **Un proceso por contenedor:** la aplicación y la base de datos van en contenedores separados.

---

## PARA ENTREGAR CON ESTE TALLER

### 1) Repositorio

- Repositorio Git con las carpetas `hello-world/` y `friendlyhello/` del taller, con **URL pública o acceso por invitación**.
- Archivo **`integrantes.txt`** o sección en el README con nombres y correos institucionales.
- La carpeta `data/` excluida del repositorio con `.gitignore`.

### 2) Documentación en Wiki (obligatoria)

> Toda la documentación del taller se entrega en el **Wiki del repositorio**.
> No se requiere PDF; el Wiki es la entrega oficial.

Estructura mínima sugerida del Wiki:

- **Inicio:** integrantes y propósito del taller.
- **Imágenes:** explicación del Dockerfile de `friendlyhello` y captura de `docker build --check` sin advertencias.
- **Caché de capas:** captura del Paso 4 con la capa de dependencias en `CACHED` y la comparación de tiempos de la pregunta de discusión.
- **Compose:** captura del contador de visitas funcionando y de `docker compose ps`.
- **Balanceo de carga:** capturas de al menos 3 *hostnames* distintos y del dashboard de Traefik.
- **Registro:** enlace a la imagen publicada en Docker Hub.
- **Proyecto:** Dockerfile *multi-stage* del proyecto del equipo y captura de la aplicación corriendo en un contenedor.
- **Reflexión final.**

### 3) Imagen de la aplicación web

- `Dockerfile`, `.dockerignore` y `requirements.txt` con versiones fijas.
- El contenedor corre con un usuario sin privilegios y aparece como `healthy`.

### 4) Docker Compose y balanceo

- `docker-compose.yaml` con *healthcheck* y `depends_on` con condición.
- `docker-compose.traefik.yaml` con 5 réplicas atendiendo peticiones.

### 5) Publicación en un registro

- Imagen publicada con una etiqueta de versión y `latest`.
- Ejercicios de la Parte 3 resueltos.

### 6) Dockerfile del proyecto

- Dockerfile *multi-stage* en el repositorio del proyecto, con usuario sin privilegios y `.dockerignore`.

### 7) Reflexión final (en el Wiki)

- ¿Qué diferencia hay entre una imagen y un contenedor? ¿Y entre un volumen y la capa de escritura del contenedor?
- ¿Por qué el orden de las instrucciones del Dockerfile afecta el tiempo de construcción?
- ¿Por qué 5 réplicas en una sola máquina no dan alta disponibilidad?
- ¿Qué riesgos de seguridad tiene una imagen que corre como `root` o que incluye herramientas de compilación?

### 8) Rúbrica – Taller de Docker

| **Criterios de evaluación** | **Indicadores de cumplimiento** | **Excelente (5 pts)** | **Bueno (4 pts)** | **Necesita mejorar (3.5 pts)** | **Deficiente (2.5 pts)** | **No cumple (0 pts)** |
|-----------------------------|----------------------------------|------------------------|-------------------|-------------------------------|--------------------------|------------------------|
| **Repositorio** | Repositorio organizado con las carpetas del taller e integrantes. | Ordenado, con `.gitignore` y archivos completos. | Completo con detalles menores de organización. | Faltan archivos o hay archivos innecesarios (por ejemplo, `data/`). | Estructura desordenada o incompleta. | No entrega. |
| **Imagen de la aplicación** | Dockerfile con buenas prácticas. | Versiones fijas, orden de capas, `.dockerignore`, `USER` y `HEALTHCHECK`; `--check` sin advertencias. | Imagen funcional con una buena práctica ausente. | Imagen funcional con varias buenas prácticas ausentes. | La imagen no construye o no ejecuta. | No hay Dockerfile. |
| **Caché de capas** | Comprensión del efecto del orden de las capas. | Evidencia de `CACHED` y comparación de tiempos con el orden alternativo. | Evidencia de `CACHED` sin comparación. | Explicación sin evidencia. | Explicación incorrecta. | No realiza el paso. |
| **Docker Compose** | Servicios web y Redis comunicándose. | Contador funcionando, *healthcheck* y `depends_on` con condición. | Contador funcionando sin *healthcheck*. | Servicios levantan pero no se comunican. | El archivo no levanta los servicios. | No hay archivo de Compose. |
| **Balanceo de carga** | Réplicas detrás de Traefik. | 5 réplicas con *hostnames* distintos evidenciados y dashboard. | Balanceo funcionando con menos evidencias. | Traefik levanta pero no distribuye tráfico. | Configuración incorrecta. | No realiza el paso. |
| **Publicación en registro** | Imagen versionada en Docker Hub. | Imagen con versión y `latest`, y ejercicios resueltos. | Imagen publicada sin versión o sin ejercicios. | Solo un ejercicio resuelto. | Intento sin imagen publicada. | No realiza la parte. |
| **Dockerfile del proyecto** | Imagen *multi-stage* del proyecto del equipo. | *Multi-stage*, usuario sin privilegios, `.dockerignore` y aplicación funcionando. | *Multi-stage* funcionando con una práctica ausente. | Imagen de una sola etapa funcionando. | La imagen no construye. | No entrega. |
| **Documentación en Wiki** | Secciones completas con evidencias. | Wiki completo, claro y con capturas de cada parte. | Wiki completo con leves omisiones. | Wiki incompleto o con poca claridad. | Wiki muy limitado o confuso. | No hay Wiki o está vacío. |
| **Reflexión técnica** | Análisis de resultados y aprendizajes. | Reflexión profunda, conectada con el proyecto. | Reflexión correcta pero superficial. | Reflexión breve o poco argumentada. | Reflexión vaga o sin relación con el taller. | No presenta reflexión. |

> **Cómo suma**: 9 criterios × 5 pts = **45 puntos**.

| Rango de puntaje | Desempeño                                                |
| ---------------- | -------------------------------------------------------- |
| 40 – 45          | Excelente dominio técnico y metodológico.                |
| 32 – 39          | Buen trabajo con evidencias o documentación parcial.     |
| 27 – 31          | Cumple con lo básico pero sin profundidad.               |
| < 27             | No cumple con los criterios mínimos del taller.          |

---

## Propósito del taller

En este taller se construyen imágenes propias a partir de una aplicación sencilla y se aplican las prácticas que exige una imagen para producción: versiones fijas, capas ordenadas, contexto de construcción limpio, usuario sin privilegios y verificación de salud.

Con Docker Compose se pasa de un contenedor a una aplicación de varios servicios, y con Traefik se ve cómo un balanceador reparte la carga entre réplicas. Finalmente, lo aprendido se aplica a la aplicación Java del proyecto, para que su imagen se pueda construir, escanear y publicar desde el pipeline de CI.

---

## Cómo usar esta guía para tu proyecto

1. **Agrega el Dockerfile *multi-stage*** de la Parte 4 a la raíz del repositorio del proyecto, con su `.dockerignore`.
2. **Escribe un `docker-compose.yaml`** con la aplicación y sus dependencias (base de datos, *broker* de mensajes), con *healthchecks* y `depends_on` con condición.
3. **Fija las versiones** de todas las imágenes del archivo de Compose.
4. **Lleva la configuración a variables de entorno**: contraseñas, *tokens* y cadenas de conexión no van en el código ni en la imagen. Agrega un `.env.example` con las variables necesarias y valores de ejemplo, y deja el `.env` real fuera del repositorio con `.gitignore`.
5. **Agrega un job de Docker al pipeline de CI** que construya la imagen solo si las pruebas pasan (`needs:`).
6. **Escanea la imagen con Trivy** y actualiza las dependencias con vulnerabilidades críticas.

Checklist para el Proyecto 3 (sección 1 del enunciado):

- [ ] Dockerfile *multi-stage* con imagen base de versión fija y usuario sin privilegios.
- [ ] `.dockerignore` en el repositorio.
- [ ] `docker-compose.yml` con el sistema y todas sus dependencias: `docker compose up` deja el sistema funcionando.
- [ ] Configuración por variables de entorno y `.env.example`, sin secretos en el código ni en la imagen.
- [ ] *Healthcheck* usado por Compose y por el despliegue.
- [ ] Imagen construida en el pipeline de CI.

---

> **Resultado esperado:**
> Al finalizar este taller, cada equipo contará con la **aplicación del proyecto contenerizada** con un Dockerfile *multi-stage*, un **archivo de Compose** que levanta la aplicación con sus dependencias y la experiencia de **escalar y balancear** un servicio, como base para el **monitoreo**, la **seguridad** y el **despliegue continuo** de la aplicación.

---

## Hagamos un resumen

### Imágenes y contenedores

- **Qué son:** la imagen es la plantilla de solo lectura; el contenedor es una instancia en ejecución de esa imagen.
- **Para qué sirven:** empaquetar la aplicación con todas sus dependencias para que se ejecute igual en cualquier equipo con Docker.
- **Ejemplo típico:** `docker build -t friendlyhello .` y `docker run -p 4000:8000 friendlyhello`.

### Dockerfile

- **Qué es:** el archivo con las instrucciones para construir una imagen, capa por capa.
- **Buenas prácticas:** versiones fijas, dependencias antes que el código, `.dockerignore`, `USER`, `HEALTHCHECK` y *multi-stage*.

### Docker Compose

- **Qué es:** la definición en YAML de una aplicación de varios servicios que comparten una red.
- **Para qué sirve:** levantar la aplicación completa con un comando y controlar el orden de arranque con *healthchecks*.

### Balanceo de carga

- **Qué es:** distribuir las peticiones entre varias réplicas de un servicio.
- **Limitación de la demostración:** en una sola máquina no hay alta disponibilidad; para eso se necesita un orquestador con varios nodos.

---

## Conclusión

En conjunto, estas prácticas permiten:

- Empaquetar una aplicación y sus dependencias en una **imagen reproducible**.
- Construir imágenes **más pequeñas, rápidas y seguras**.
- Levantar aplicaciones de **varios servicios** con un solo comando.
- **Escalar** un servicio y repartir la carga entre réplicas.
- Distribuir imágenes a través de un **registro**.

Con esto se obtiene una base común para ejecutar, probar y desplegar la aplicación en cualquier entorno, que es el punto de partida del despliegue continuo.

---

## Recursos recomendados

- Documentación oficial de Docker – [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- Documentación oficial de Docker – [Building best practices](https://docs.docker.com/build/building/best-practices/)
- Documentación oficial de Docker – [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- Documentación oficial de Docker – [Docker Compose](https://docs.docker.com/compose/)
- [Traefik – Docker provider](https://doc.traefik.io/traefik/providers/docker/)
- [Trivy](https://trivy.dev/)
- Aula de Software Libre (Universidad de Córdoba) – [Taller de Docker](https://aulasoftwarelibre.github.io/taller-de-docker/)

---

## Créditos y uso académico

**Autor:** César Augusto Vega Fernández
**Curso:** Diseño y Arquitectura de Software
**Programa:** Ingeniería Informática – Universidad de La Sabana
**Año:** 2026

Este taller y su contenido fueron preparados por el profesor **César Augusto Vega Fernández** como material académico para el curso *Diseño y Arquitectura de Software*, impartido en el programa de **Ingeniería Informática de la Universidad de La Sabana**.

Las Partes 1 a 3 están basadas en el [Taller de Docker](https://aulasoftwarelibre.github.io/taller-de-docker/) de **Aula de Software Libre** (Universidad de Córdoba), distribuido bajo licencia [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/deed.es). Sobre ese material se actualizaron los comandos y salidas a BuildKit y Docker Compose v2, se aplicaron buenas prácticas de construcción de imágenes, se reemplazó la versión de Traefik y se agregaron la Parte 4, el escaneo de vulnerabilidades, los problemas frecuentes y los entregables.

Su propósito es exclusivamente educativo y está orientado a fortalecer las competencias de los estudiantes en **contenerización, construcción de imágenes, orquestación local con Docker Compose** y prácticas **DevOps** aplicadas al desarrollo de software.

---

### Licencia de uso

Este material se distribuye bajo la licencia [Creative Commons Atribución-NoComercial-CompartirIgual 4.0 Internacional (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.es).

Puedes **usar, adaptar o compartir** este contenido con fines educativos, siempre que:

1. Se reconozca la autoría del profesor **César Augusto Vega Fernández** y de **Aula de Software Libre** en las partes basadas en su taller.
2. No se utilice con fines comerciales.
3. Las obras derivadas se distribuyan bajo la misma licencia.

---

© Universidad de La Sabana – Facultad de Ingeniería
Ingeniería Informática – 2026
