# Understanding `compose.yml`: A Practical Guide

This guide explains the structure of a Docker Compose file, what its
main elements mean, and how to use them in a working example.

## Table of contents

-   [1. What is `compose.yml`?](#1-what-is-composeyml)
-   [2. A minimal example](#2-a-minimal-example)
-   [3. The structure of a Compose
    file](#3-the-structure-of-a-compose-file)
-   [4. `services`](#4-services)
-   [5. `image` and `build`](#5-image-and-build)
-   [6. `ports`](#6-ports)
-   [7. `environment` and `.env`](#7-environment-and-env)
-   [8. `volumes`](#8-volumes)
-   [9. `networks`](#9-networks)
-   [10. `depends_on` and health
    checks](#10-depends_on-and-health-checks)
-   [11. `command` and `entrypoint`](#11-command-and-entrypoint)
-   [12. Restart policies](#12-restart-policies)
-   [13. Other useful options](#13-other-useful-options)
-   [14. A complete example](#14-a-complete-example)
-   [15. Common commands](#15-common-commands)
-   [16. Common mistakes](#16-common-mistakes)
-   [17. Quick reference](#17-quick-reference)

------------------------------------------------------------------------

## 1. What is `compose.yml`?

`compose.yml` is a YAML configuration file used by Docker Compose to
define the services that make up an application and how they run
together.

Instead of creating containers and configuring networks, ports, and
storage one at a time, describe the desired setup in this file and run:

``` bash
docker compose up -d
```

Docker Compose reads the configuration and creates or updates the
required resources.

Compose files are also commonly named `compose.yaml`,
`docker-compose.yml`, or `docker-compose.yaml`. If you use a non-default
filename or location, pass it with `-f`.

## 2. A minimal example

``` yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
```

Run it:

``` bash
docker compose up -d
```

Open <http://localhost:8080>.

What the example means:

-   `services`: declares the application services.
-   `web`: the service name chosen by you.
-   `image`: the container image Docker should use.
-   `ports`: publishes container port `80` on host port `8080`.

To stop and remove the container and the default network:

``` bash
docker compose down
```

## 3. The structure of a Compose file

A Compose file can include several top-level sections:

``` yaml
services:
  # Application containers go here

volumes:
  # Named persistent storage goes here

networks:
  # Custom networks go here

configs:
  # Optional configuration objects

secrets:
  # Optional sensitive configuration objects
```

You only need the sections your application uses. For example, a simple
Nginx container may need only `services`. A database-backed application
may need `services` and `volumes`.

The `services` section is central: each service defines how a container
or group of replicated containers should be configured.

## 4. `services`

A **service** is a named definition for a type of container in your
application. Common services include `web`, `api`, `db`, `redis`, and
`worker`.

``` yaml
services:
  web:
    image: nginx:alpine

  redis:
    image: redis:7-alpine
```

In this example, Compose defines two services: `web` and `redis`.

A service is not exactly the same thing as a single container. Compose
can create one or more containers for a service, depending on
configuration such as scaling.

### Why use services?

-   Group related parts of an application in one file.
-   Start and stop related components together.
-   Give containers service names for network communication.
-   Configure each component independently.

### Service names are useful hostnames

If a service is named `redis`, another service on the same Compose
network can connect to it using hostname `redis` and Redis's internal
port `6379`.

Use the **container port** for service-to-service traffic, not the
host-published port.

## 5. `image` and `build`

These tell Compose where a service's container image comes from.

### `image`

Use an existing image from a registry or one already available locally:

``` yaml
services:
  db:
    image: postgres:16
```

The image name is `postgres` and the tag is `16`.

Compose pulls the image when needed. Prefer an intentional version tag
instead of `latest` for more predictable environments.

### `build`

Build an image from a Dockerfile:

``` yaml
services:
  api:
    build: .
```

Here, `.` is the build context: the current directory. By default,
Docker looks for a file named `Dockerfile` in that context.

Example project:

``` text
my-project/
├── compose.yml
├── Dockerfile
└── app.py
```

A basic Dockerfile could be:

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

Compose can build and start it:

``` bash
docker compose up -d --build
```

### `build` with options

``` yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
```

-   `context`: files available to the build.
-   `dockerfile`: the Dockerfile path relative to the build context,
    unless otherwise specified by the supported build configuration.

### Can `image` and `build` appear together?

Yes. For example:

``` yaml
services:
  api:
    image: my-api:dev
    build: .
```

This defines a build for the service and names the resulting image
`my-api:dev`. Build and pull behavior can depend on Compose
configuration and command options.

## 6. `ports`

`ports` publishes a container port to the host, allowing applications
outside the container network---such as your browser---to reach the
service.

``` yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
```

The format is generally:

``` text
HOST_PORT:CONTAINER_PORT
```

For `"8080:80"`:

-   `8080` is the port on your computer.
-   `80` is the port inside the container.
-   Open `http://localhost:8080` in your browser.

### Different host port

If host port `8080` is already in use, use another host port:

``` yaml
ports:
  - "8081:80"
```

Then open `http://localhost:8081`.

### Bind to localhost

For a development service you only want reachable from the same machine:

``` yaml
ports:
  - "127.0.0.1:8080:80"
```

Without a host IP, a published port is typically exposed on all host
network interfaces. Consider the security implications before publishing
services on a network.

### `ports` versus `expose`

-   `ports` publishes a port to the host.
-   `expose` documents or makes a port available to other services on
    the container network, but does not publish it to the host.

You usually do not need `expose` for ordinary Compose networking:
services on the same network can communicate using the container port.

## 7. `environment` and `.env`

Environment variables let you configure an application without
hard-coding settings into its image.

``` yaml
services:
  api:
    image: my-api:dev
    environment:
      APP_MODE: development
      LOG_LEVEL: debug
```

The application can read `APP_MODE` and `LOG_LEVEL` from its
environment.

### Use a `.env` file for Compose interpolation

Create a `.env` file beside `compose.yml`:

``` dotenv
WEB_PORT=8080
```

Use the variable in Compose:

``` yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "${WEB_PORT:-8080}:80"
```

Compose substitutes the variable when it reads the file.
`${WEB_PORT:-8080}` means to use `8080` if `WEB_PORT` is unset or empty.

**Important distinction:**

-   A `.env` file is commonly used to supply values for substitution in
    the Compose file.
-   The `environment` section passes specified environment variables
    into the container.
-   A variable in `.env` is not automatically available inside every
    container unless you pass it through configuration, such as
    `environment` or `env_file`.

You can also use `env_file`:

``` yaml
services:
  api:
    image: my-api:dev
    env_file:
      - app.env
```

This passes variables from `app.env` into the container environment.

**Security:** Do not commit real passwords, API keys, or production
secrets to source control. A `.env` file is not a secrets vault. Keep
private files out of Git and use a suitable secrets-management approach
for sensitive values.

## 8. `volumes`

Volumes are used to persist data or make files available to containers.

### Why are volumes important?

A container's writable layer is tied to that container. When a container
is removed and recreated, data stored only in that layer may be lost. A
volume lets data live independently of a particular container.

### A. Named volume

``` yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

-   `db_data` is the named volume managed by Docker.
-   `/var/lib/postgresql/data` is the path inside the PostgreSQL
    container where database data is stored.

Compose creates the named volume if necessary.

### B. Bind mount

A bind mount maps a path from your computer into the container:

``` yaml
services:
  web:
    image: nginx:alpine
    volumes:
      - ./html:/usr/share/nginx/html:ro
```

-   `./html` is a directory relative to the Compose project directory.
-   `/usr/share/nginx/html` is the path inside the container.
-   `ro` means read-only from the container's perspective.

Bind mounts are useful during development when you want to edit local
files and see changes inside a container.

### Named volumes versus bind mounts

  -----------------------------------------------------------------------
  Feature                 Named volume            Bind mount
  ----------------------- ----------------------- -----------------------
  Source                  Managed by Docker       Specific host file or
                                                  directory

  Common use              Database and persistent Source code and local
                          application data        configuration

  Host path               Docker manages its      You choose the host
                          storage location        path

  Portability             Often easier to manage  Depends on local file
                          across machines         paths
  -----------------------------------------------------------------------

### Important data warning

`docker compose down` normally leaves named volumes intact. The command
below also removes named volumes declared in the Compose project and can
delete stored database data:

``` bash
docker compose down -v
```

Use it only when you intentionally want to remove that data. A volume is
not a substitute for a backup.

## 9. `networks`

Networks let services communicate with one another.

Compose normally creates a default network for the project. Services
attached to the same network can discover one another using service
names.

``` yaml
services:
  api:
    image: my-api:dev
    networks:
      - app_network

  db:
    image: postgres:16
    networks:
      - app_network

networks:
  app_network:
```

The `api` service can connect to PostgreSQL at hostname `db` and its
internal port `5432`, provided the application has the appropriate
database credentials and configuration.

### Why define a custom network?

-   Separate groups of services when needed.
-   Control which services share network connectivity.
-   Make a larger configuration easier to understand.

For simple projects, the default network is often enough. Avoid
publishing database ports to the host unless you actually need host-side
access.

## 10. `depends_on` and health checks

### `depends_on`

`depends_on` expresses a dependency between services.

``` yaml
services:
  api:
    image: my-api:dev
    depends_on:
      - db

  db:
    image: postgres:16
```

This controls the startup order: Compose starts `db` before `api`.
However, a running database container may not yet be ready to accept
connections.

### Wait for a health check

For a database that supports a health check:

``` yaml
services:
  api:
    image: my-api:dev
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: localdevpassword
      POSTGRES_DB: appdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 10
```

Key health-check fields:

-   `test`: command used to check whether the service is healthy.
-   `interval`: time between checks.
-   `timeout`: maximum time for one check.
-   `retries`: number of consecutive failures before marking the
    container unhealthy.

`condition: service_healthy` asks Compose to wait for the dependency's
health check to pass before starting the dependent service.

A health check is only as good as its test. Also make the application
retry connections when appropriate: a dependency can become unavailable
after startup.

## 11. `command` and `entrypoint`

These control what runs inside a container.

### `command`

Overrides the image's default command:

``` yaml
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
```

This starts Redis with append-only persistence enabled.

### `entrypoint`

Overrides the image's configured entrypoint:

``` yaml
services:
  worker:
    image: python:3.12-slim
    entrypoint: ["python"]
    command: ["-c", "print('Worker started')"]
```

The container runs Python with the provided command.

In practice, `command` and `entrypoint` interact with the defaults
defined by the image's Dockerfile. Check the image documentation before
overriding them.

## 12. Restart policies

A restart policy tells Docker when to restart a container.

``` yaml
services:
  api:
    image: my-api:dev
    restart: unless-stopped
```

Common values:

-   `no`: do not automatically restart; this is the default.
-   `always`: restart the container whenever it stops, subject to
    Docker's restart behavior.
-   `on-failure`: restart when the process exits with a non-zero status;
    an optional retry limit can be specified.
-   `unless-stopped`: restart unless the container was explicitly
    stopped.

For local development, you may not need a restart policy. For
long-running services, choose one that fits your operational needs.

A restart policy does not fix a broken application. Use logs and health
checks to diagnose repeated failures.

## 13. Other useful options

### `working_dir`

Sets the working directory inside the container:

``` yaml
services:
  api:
    image: python:3.12-slim
    working_dir: /app
```

### `user`

Specifies the user used to run the process, where supported by the
image:

``` yaml
services:
  app:
    image: my-app:dev
    user: "1000:1000"
```

Check file ownership and permissions before changing the user.

### `read_only`

Makes the container's root filesystem read-only:

``` yaml
services:
  app:
    image: my-app:dev
    read_only: true
```

Some applications need writable temporary directories or other writable
mounts, so test before enabling this.

### `profiles`

Profiles let you enable optional services only when requested:

``` yaml
services:
  web:
    image: nginx:alpine

  debug:
    image: busybox
    command: ["sh", "-c", "echo Debug service started; sleep 3600"]
    profiles:
      - debug
```

Start the default services:

``` bash
docker compose up -d
```

Start the optional profile too:

``` bash
docker compose --profile debug up -d
```

### `container_name`

Sets a fixed container name:

``` yaml
services:
  web:
    image: nginx:alpine
    container_name: my-nginx
```

Usually, you should omit `container_name` unless you have a specific
reason to set it. Fixed names can make scaling and running multiple
copies of a project harder.

### `logging`

Logging configuration can control how Docker collects and rotates
container logs. The exact options depend on the logging driver and
environment.

``` yaml
services:
  web:
    image: nginx:alpine
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### `secrets`

Compose supports declaring and granting access to secrets:

``` yaml
services:
  app:
    image: my-app:dev
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

This makes the declared secret available to the service using the
Compose secrets mechanism. How secrets are stored and protected depends
on the Compose implementation and deployment environment. Protect the
source file and do not commit it to Git.

## 14. A complete example

This example combines a Python web service, PostgreSQL, a named volume,
environment variables, port publishing, a health check, and a
dependency.

Project structure:

``` text
my-project/
├── compose.yml
├── Dockerfile
├── app.py
├── requirements.txt
└── .env
```

### `compose.yml`

``` yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "${WEB_PORT:-8000}:8000"
    environment:
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: ${POSTGRES_DB:-appdb}
      DB_USER: ${POSTGRES_USER:-appuser}
      DB_PASSWORD: ${POSTGRES_PASSWORD:-localdevpassword}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-appdb}
      POSTGRES_USER: ${POSTGRES_USER:-appuser}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-localdevpassword}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U \"$${POSTGRES_USER}\" -d \"$${POSTGRES_DB}\""]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

volumes:
  postgres_data:
```

### `.env`

``` dotenv
WEB_PORT=8000
POSTGRES_DB=appdb
POSTGRES_USER=appuser
POSTGRES_PASSWORD=localdevpassword
```

This password is only for local learning. Use secure secret management
for real deployments. Do not commit a real `.env` file; add it to
`.gitignore`.

### `Dockerfile`

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "app.py"]
```

### `requirements.txt`

``` text
Flask>=3.0,<4.0
psycopg[binary]>=3.1,<4.0
```

### `app.py`

This small example connects to PostgreSQL and returns a response:

``` python
import os
import time
from flask import Flask
import psycopg
from psycopg import OperationalError

app = Flask(__name__)

def connect_db():
    return psycopg.connect(
        host=os.environ["DB_HOST"],
        port=os.environ["DB_PORT"],
        dbname=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
        connect_timeout=3,
    )

@app.get("/")
def home():
    for attempt in range(10):
        try:
            with connect_db() as conn:
                with conn.cursor() as cur:
                    cur.execute("SELECT version()")
                    version = cur.fetchone()[0]
            return f"Connected to PostgreSQL!\\n{version}\\n"
        except OperationalError:
            if attempt == 9:
                raise
            time.sleep(2)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

Run from the project directory:

``` bash
docker compose config
docker compose up -d --build
docker compose ps
docker compose logs -f
```

Open <http://localhost:8000>.

Stop the application:

``` bash
docker compose down
```

The named volume `postgres_data` remains by default. To remove it too,
use `docker compose down -v`---only if you are sure you want to delete
the stored database data.

This is a learning example, not a production-ready application. For
production, use a production WSGI server, secure secrets, pinned
dependency versions, proper database migrations, and appropriate
monitoring and backup procedures.

## 15. Common commands

Run commands from the directory containing your Compose file.

  -----------------------------------------------------------------------
  Command                             What it does
  ----------------------------------- -----------------------------------
  `docker compose up`                 Creates and starts services in the
                                      foreground

  `docker compose up -d`              Starts services in the background

  `docker compose up -d --build`      Builds configured images and starts
                                      services

  `docker compose down`               Stops and removes containers and
                                      the default network

  `docker compose down -v`            Also removes declared named and
                                      attached anonymous volumes; may
                                      delete data

  `docker compose ps`                 Lists the project's containers and
                                      status

  `docker compose logs -f`            Follows service logs

  `docker compose logs -f web`        Follows logs for the `web` service

  `docker compose exec web sh`        Opens a shell in a running service
                                      container, if available

  `docker compose restart`            Restarts services

  `docker compose stop`               Stops services without removing
                                      them

  `docker compose start`              Starts previously created, stopped
                                      services

  `docker compose pull`               Pulls configured service images

  `docker compose build`              Builds services configured with
                                      `build:`

  `docker compose config`             Validates and renders the resolved
                                      Compose model
  -----------------------------------------------------------------------

For a file with a different name:

``` bash
docker compose -f compose.dev.yml up -d
```

## 16. Common mistakes

### Wrong YAML indentation

YAML uses spaces and indentation to express nesting. Use spaces, not
tabs.

Correct:

``` yaml
services:
  web:
    image: nginx:alpine
```

### Confusing host and container ports

In `"8080:80"`, the left side is the host port and the right side is the
container port.

### Using `localhost` to reach another service

Inside a container, `localhost` means that same container. For a
database service named `db`, use `db:5432` from another service on the
same network.

### Assuming `depends_on` means the database is ready

Short-form `depends_on` provides startup ordering but does not wait for
the service to become healthy. Use a health check and
`condition: service_healthy` where appropriate, and implement connection
retries in the application.

### Expecting `ports` for service-to-service communication

Services on the same Compose network can communicate using service names
and container ports without publishing those ports to the host.

### Losing data with `down -v`

`docker compose down -v` removes named volumes declared in the project
and can delete database data. Use it carefully.

### Forgetting to rebuild after code or dependency changes

When the app is copied into the image at build time, rebuild after
changes:

``` bash
docker compose up -d --build
```

For faster development, a bind mount can make local source files
available inside the container, but configure it deliberately.

## 17. Quick reference

  -----------------------------------------------------------------------
  Element                             Meaning
  ----------------------------------- -----------------------------------
  `services`                          Defines the application's services

  `image`                             Selects an existing image

  `build`                             Builds an image from a Dockerfile
                                      and build context

  `ports`                             Publishes container ports to the
                                      host

  `environment`                       Sets environment variables inside a
                                      container

  `env_file`                          Loads environment variables from a
                                      file into a container

  `.env`                              Common source of values used for
                                      Compose variable substitution

  `volumes`                           Defines persistent storage or
                                      mounts files/directories

  `networks`                          Defines networks used by services

  `depends_on`                        Expresses service startup
                                      dependencies

  `healthcheck`                       Tests whether a container's service
                                      is healthy

  `command`                           Overrides the image's default
                                      command

  `entrypoint`                        Overrides the image's entrypoint

  `restart`                           Sets the container restart policy

  `working_dir`                       Sets the working directory inside
                                      the container

  `profiles`                          Enables optional services for
                                      selected workflows

  `secrets`                           Declares sensitive values and
                                      grants services access to them

  `configs`                           Declares configuration data for
                                      services, where supported
  -----------------------------------------------------------------------

**Remember:** Start by learning `services`, `image`/`build`, `ports`,
`environment`, and `volumes`. These cover much of what you'll use in
everyday development. Add networks, health checks, profiles, secrets,
and other settings as your application needs them.
