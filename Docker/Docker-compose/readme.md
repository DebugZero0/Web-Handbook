# Docker Compose: Purpose, Benefits, and Practical Examples

Docker Compose is a tool for defining and running multi-container Docker
applications. You describe the services your application needs in a YAML
file---commonly `compose.yml`---and manage them together using the
`docker compose` command.

Instead of manually creating each container, network, and volume, you
can configure them once and run:

``` bash
docker compose up -d
```

> **Prerequisites:** Install Docker Desktop (Windows/macOS) or Docker
> Engine plus the Docker Compose plugin (Linux). Verify with
> `docker --version` and `docker compose version`.

## Table of contents

1.  [Manage multiple containers
    together](#1-manage-multiple-containers-together)
2.  [Start and stop the application with simple
    commands](#2-start-and-stop-the-application-with-simple-commands)
3.  [Let services communicate over a
    network](#3-let-services-communicate-over-a-network)
4.  [Centralize configuration](#4-centralize-configuration)
5.  [Persist data with volumes](#5-persist-data-with-volumes)
6.  [Build your own application
    image](#6-build-your-own-application-image)
7.  [Make the environment
    reproducible](#7-make-the-environment-reproducible)
8.  [Useful commands](#8-useful-docker-compose-commands)
9.  [A small complete application](#9-a-small-complete-application)
10. [Troubleshooting](#10-troubleshooting)

------------------------------------------------------------------------

## 1. Manage multiple containers together

**Purpose:** A real application often needs several services, such as a
web server and a database. Compose lets you define them in one file.

### Example: Web server and database

Create `compose.yml`:

``` yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: localdevpassword
```

Start both services:

``` bash
docker compose up -d
```

Check them:

``` bash
docker compose ps
```

Open <http://localhost:8080> to see the Nginx welcome page.

**How it helps:** One Compose file describes the related services. You
don't need to create and configure each container manually.

> The password in this introductory example is for local learning only.
> Use secrets or a secure environment configuration for real
> deployments, and never commit production credentials to source
> control.

## 2. Start and stop the application with simple commands

**Purpose:** Use one command to start the project, and another to stop
and remove its containers and default network.

Using the Compose file from the previous example:

``` bash
# Start services in the background
docker compose up -d

# See service status
docker compose ps

# Follow logs
docker compose logs -f

# Stop and remove containers and the default network
docker compose down
```

To stop the services without removing their containers:

``` bash
docker compose stop
```

**How it helps:** You don't need to remember separate `docker run`,
`docker stop`, and network commands for every service.

`docker compose down` does **not** normally delete named volumes. To
remove named volumes too, you would explicitly use
`docker compose down -v`; be careful because that can delete stored
data.

## 3. Let services communicate over a network

**Purpose:** Compose creates a project network by default. Services can
usually reach one another using their service names as hostnames.

### Example: An API connecting to PostgreSQL

``` yaml
services:
  api:
    image: python:3.12-slim
    command: python -c "import time; print('Example API container started'); time.sleep(3600)"
    environment:
      DB_HOST: db
      DB_PORT: "5432"
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: localdevpassword
```

Start the services:

``` bash
docker compose up -d
docker compose logs api
```

The API's database hostname is `db`, because `db` is the service name.
Inside the Compose network, PostgreSQL listens on port `5432`.

**How it helps:** You normally don't need to hard-code container IP
addresses. Compose service names provide stable names for
service-to-service connections.

**Important:** `depends_on` controls startup order, but by itself it
does not guarantee that PostgreSQL is ready to accept connections. For
robust applications, add a database health check and configure the
application to retry connections.

## 4. Centralize configuration

**Purpose:** Keep ports, environment variables, and service settings
together instead of scattering them across command-line arguments.

### Example: Environment variables

``` yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "${WEB_PORT:-8080}:80"
```

Create a `.env` file beside `compose.yml`:

``` dotenv
WEB_PORT=8088
```

Start Compose:

``` bash
docker compose up -d
```

Now open <http://localhost:8088>.

If `WEB_PORT` is not set, the `${WEB_PORT:-8080}` expression uses port
`8080`.

**How it helps:** Developers can adjust local settings without editing
the Compose file itself.

**Security note:** A `.env` file is convenient for local configuration,
but it is not a secure secrets vault. Don't commit real credentials to
Git. Add `.env` to `.gitignore` when it contains private values, and
provide a safe `.env.example` for teammates.

## 5. Persist data with volumes

**Purpose:** A container can be replaced during updates or recreation. A
named volume stores database files outside the writable layer of that
container.

### Example: PostgreSQL with a named volume

``` yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: localdevpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Start it:

``` bash
docker compose up -d
```

You can recreate the container without intentionally deleting the named
volume:

``` bash
docker compose up -d --force-recreate db
```

**How it helps:** Database files can survive container recreation
because they are stored in the `postgres_data` volume.

**Important:** A volume is not a backup. Back up important data
separately. Running `docker compose down -v` removes the named volume
and can delete the database data.

## 6. Build your own application image

**Purpose:** Compose can build an image from a `Dockerfile` and then run
a container from that image. You don't have to run `docker build`
separately for the normal workflow.

Create this structure:

``` text
my-app/
├── compose.yml
├── Dockerfile
└── app.py
```

`app.py`:

``` python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        body = b"Hello from my Docker Compose app!\n"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

`Dockerfile`:

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

`compose.yml`:

``` yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
```

Run it:

``` bash
docker compose up -d --build
```

Visit <http://localhost:8000>.

**How it helps:** The `build: .` setting tells Compose to build an image
using the Dockerfile in the current directory. The `--build` option asks
Compose to build before starting the service.

## 7. Make the environment reproducible

**Purpose:** Put the environment definition in version control so
teammates can start the same set of services without manually
reconstructing every `docker run` command.

A simple project might contain:

``` text
project/
├── compose.yml
├── Dockerfile
├── app.py
├── .env.example
└── .gitignore
```

Example `.gitignore`:

``` gitignore
.env
__pycache__/
```

Example `.env.example`:

``` dotenv
WEB_PORT=8000
```

A teammate can clone the project, create their own `.env` if required,
and run:

``` bash
docker compose up -d --build
```

**How it helps:** The team shares the service definitions, port
mappings, build instructions, and volume configuration. This improves
consistency between developer machines.

**Note:** Reproducibility also depends on image versions, application
dependencies, operating-system differences, and external services. For
more predictable builds, pin image versions and lock application
dependencies.

## 8. Useful Docker Compose commands

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `docker compose up -d`              Create and start services in the
                                      background

  `docker compose up -d --build`      Build configured images, then start
                                      services

  `docker compose ps`                 Show services and their status

  `docker compose logs -f`            Follow logs from all services

  `docker compose logs -f api`        Follow logs for a service named
                                      `api`

  `docker compose exec api sh`        Open a shell in a running `api`
                                      container, if it has `sh`

  `docker compose restart`            Restart services

  `docker compose stop`               Stop services without removing
                                      containers

  `docker compose start`              Start previously stopped containers

  `docker compose down`               Stop and remove containers and the
                                      default network

  `docker compose config`             Validate and render the resolved
                                      Compose configuration

  `docker compose pull`               Pull configured images

  `docker compose build`              Build services configured with
                                      `build:`
  -----------------------------------------------------------------------

Run these commands from the directory containing `compose.yml`. If the
file has a different name or is elsewhere, specify it with `-f`, for
example:

``` bash
docker compose -f path/to/compose.yml up -d
```

## 9. A small complete application

This final example combines a custom Python web application with Redis.
The web app displays a visit counter stored in Redis, demonstrating a
multi-service application and service-name networking.

Project structure:

``` text
compose-counter/
├── compose.yml
├── Dockerfile
├── requirements.txt
└── app.py
```

### `app.py`

``` python
import os
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis_client = Redis(
    host=os.getenv("REDIS_HOST", "redis"),
    port=6379,
    decode_responses=True,
    socket_connect_timeout=2,
    socket_timeout=2,
)

@app.get("/")
def home():
    visits = redis_client.incr("visits")
    return f"Hello! This page has been visited {visits} times.\n"

@app.get("/health")
def health():
    redis_client.ping()
    return {"status": "ok"}
```

### `requirements.txt`

``` text
Flask>=3.0,<4.0
redis>=5.0,<7.0
```

For a production application, pin exact tested dependency versions in a
lock file.

### `Dockerfile`

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["flask", "--app", "app", "run", "--host=0.0.0.0", "--port=5000"]
```

### `compose.yml`

``` yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      REDIS_HOST: redis
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    command: ["redis-server", "--appendonly", "yes"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  redis_data:
```

### Run the complete application

From the `compose-counter` directory:

``` bash
docker compose up -d --build
```

Open <http://localhost:5000>. Refresh the page to increment the counter.
Redis stores the counter, and the named volume keeps Redis's persisted
data across container recreation.

Useful commands for this project:

``` bash
docker compose ps
docker compose logs -f
docker compose down
```

This is a learning example. Flask's development server is not intended
for production deployment; use a production WSGI server and appropriate
security configuration for a real application.

## 10. Troubleshooting

### Port is already in use

If port `5000` is occupied, change the mapping:

``` yaml
ports:
  - "5001:5000"
```

Then open `http://localhost:5001`.

### Check the resolved configuration

``` bash
docker compose config
```

This helps identify malformed YAML and configuration issues.

### View logs for a failing service

``` bash
docker compose logs web
docker compose logs redis
```

### Rebuild after changing the Dockerfile or dependencies

``` bash
docker compose up -d --build
```

### Confirm Docker Compose is available

``` bash
docker --version
docker compose version
```

------------------------------------------------------------------------

## Summary

Docker Compose is useful because it lets you:

-   Define multiple services in one YAML file.
-   Start and stop a complete application with a few commands.
-   Connect services using service names.
-   Manage ports, environment variables, and volumes centrally.
-   Build custom images and start containers in one workflow.
-   Share a consistent development setup with a team.

**The key idea:** Write the service configuration once in `compose.yml`,
then use `docker compose up` to create and run the application.
